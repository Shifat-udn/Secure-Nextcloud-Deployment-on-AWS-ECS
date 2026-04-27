## Web Application Firewall - Troubleshoot

On the WAF ACL section we have defined log location  

<p align="center">
  <img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/troubleshoot-WAF .png" />
</p>

now go to Amazon CloudWatch and find the location (i.e. /aws-waf-logs-xxxx ) and Analyze logs using filters such as:
-	%BLOCK% → to identify blocked requests
-	Rule IDs → to understand which rule triggered the action
This helps distinguish between legitimate threats and false positives.

## Scopedown - WAF rules 

To handle these false positives without weakening overall protection, a scope-down rule is implemented. The logic ensures that managed rules are applied except for specific known-safe scenarios:
<p align="center">
  <img src="https://github.com/Shifat-udn/Secure-Nextcloud-Deployment-on-AWS-ECS/blob/main/images/WAF-scopedown.png" />
</p>
for examlpe this scenario we need to put this rules 

``` sh
NOT (
((URI starts with /remote.php/dav/ ) AND (HTTP method = PROPFIND)) 
OR 
((URI starts with /apps/theming/ ) AND (HTTP method = POST))
)
goes like this 

```
json
 
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
