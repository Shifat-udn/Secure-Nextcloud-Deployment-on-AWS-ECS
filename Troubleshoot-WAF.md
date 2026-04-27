## Web Application Firewall - Troubleshoot

On the task defination we have defined log location  

```sh

            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "/ecs/ESC-Next-Cloud",
                    "awslogs-create-group": "true",
                    "awslogs-region": "us-east-1",
                    "awslogs-stream-prefix": "mra"
                },
                "secretOptions": []
            }
```
now go to Amazon CloudWatch and find the location (i.e. /ecs/ESC-Next-Cloud) and Analyze logs using filters such as:
-	%BLOCK% → to identify blocked requests
-	Rule IDs → to understand which rule triggered the action
This helps distinguish between legitimate threats and false positives.

## Scopedown - WAF rules 

To handle these false positives without weakening overall protection, a scope-down rule is implemented. The logic ensures that managed rules are applied except for specific known-safe scenarios:
<p align="center">
  <img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/WAF-scopedown.png" />
</p>
for examlpe this scenario we need to put this rules 

NOT (
((URI starts with /remote.php/dav/ ) AND (HTTP method = PROPFIND)) 
OR 
((URI starts with /apps/theming/ ) AND (HTTP method = POST))
)
goes like this 

``` sh
{
  "NotStatement": {
    "Statement": {
      "OrStatement": {
        "Statements": [
          {
            "AndStatement": {
              "Statements": [
                {
                  "ByteMatchStatement": {
                    "SearchString": "/remote.php/dav/",
                    "FieldToMatch": {
                      "UriPath": {}
                    },
                    "TextTransformations": [
                      {
                        "Priority": 0,
                        "Type": "NONE"
                      }
                    ],
                    "PositionalConstraint": "STARTS_WITH"
                  }
                },
                {
                  "ByteMatchStatement": {
                    "SearchString": "PROPFIND",
                    "FieldToMatch": {
                      "Method": {}
                    },
                    "TextTransformations": [
                      {
                        "Priority": 0,
                        "Type": "NONE"
                      }
                    ],
                    "PositionalConstraint": "EXACTLY"
                  }
                }
              ]
            }
          },
          {
            "AndStatement": {
              "Statements": [
                {
                  "ByteMatchStatement": {
                    "SearchString": "/apps/theming/",
                    "FieldToMatch": {
                      "UriPath": {}
                    },
                    "TextTransformations": [
                      {
                        "Priority": 0,
                        "Type": "NONE"
                      }
                    ],
                    "PositionalConstraint": "STARTS_WITH"
                  }
                },
                {
                  "ByteMatchStatement": {
                    "SearchString": "POST",
                    "FieldToMatch": {
                      "Method": {}
                    },
                    "TextTransformations": [
                      {
                        "Priority": 0,
                        "Type": "NONE"
                      }
                    ],
                    "PositionalConstraint": "EXACTLY"
                  }
                }
              ]
            }
          }
        ]
      }
    }
  }
}
```
