# Azure Function Proxies


## Visual Studio

- Create an Azure Function Solution
- Create an HttpTrigger function called Function1
- Add a proxies.json file to the Azure Function project
> **Note:** Click the proxies.json file and in properties make sure to change the setting ```Copy to Output Directory = Copy Always```

## Modify the Function1 code to show the headers

```c#
using System;
using System.IO;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Azure.WebJobs;
using Microsoft.Azure.WebJobs.Extensions.Http;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.Logging;
using Newtonsoft.Json;

namespace TestHttpFunc
{
    public static class Function1
    {
        [FunctionName("Function1")]
        public static async Task<IActionResult> Run(
            [HttpTrigger(AuthorizationLevel.Function, "get", "post", Route = null)] HttpRequest req,
            ILogger log)
        {
            log.LogInformation("C# HTTP trigger function processed a request.");

            string name = req.Query["name"];

            string requestBody = await new StreamReader(req.Body).ReadToEndAsync();
            dynamic data = JsonConvert.DeserializeObject(requestBody);
            name = name ?? data?.name;

            // Show the headers
            foreach(var header in req.Headers)
            {
                log.LogInformation($"{header.Key}:{header.Value}");
            }

            string responseMessage = string.IsNullOrEmpty(name)
                ? "This HTTP triggered function executed successfully. Pass a name in the query string or in the request body for a personalized response."
                : $"Hello, {name}. This HTTP triggered function executed successfully.";

            return new OkObjectResult(responseMessage);
        }
    }
}
```

## Add the following contents to the proxies.json file

- This code creates two proxies
  - Proxy1: Adds the ```x-functions-x``` with value ```Test``` and sends it to forwards it to ```https://{}/api/Function1```
  - Proxy2: Adds the ```x-functions-x``` with value ```%ANOTHERAPP_API_KEY%``` that comes from the application settings, and sends it to forwards it to ```https://www.microsoft.com```.
  - > **Note:** the URL for Proxy 2 should be set to your test external function expecting the custom header

```json
{
	"$schema": "http://json.schemastore.org/proxies",
	"proxies": {
		"proxy1": {
			"matchCondition": {
				"methods": [ "GET" ],
				"route": "/api/test1"
			},
			"backendUri": "http://localhost:7071/api/Function1",
			"requestOverrides": {
				"backend.request.headers.x-functions-key": "Test"
			}
		},
		"proxy2": {
			"matchCondition": {
				"methods": [ "GET" ],
				"route": "/api/test2"
			},
			"backendUri": "https://www.microsoft.com",
			"requestOverrides": {
				"backend.request.headers.x-functions-key": "%ANOTHERAPP_API_KEY%"				
			}
		}
	}
}
```

## Testing

- Run the Azure function and for each run in the debug console notice the headers
- Curl: http://localhost:7071/api/Function1 
> **Note:** Should not include the ```x-functions-key``` header
- Curl http://localhost:7071/api/test1 
> **Note:** Should include the ```x-functions-key``` header with value ```test```

To test proxy ```test2```, you will need to point the backend URL to a site you where you can test receiving the custom header.
