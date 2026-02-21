# Azure Function App Infrastructure - Overview <br/> Runtime and Storage Insights (Temp Files)

-----------------------------

> Contains two scenarios to compare temp file decay behavior in Azure Functions. The goal is to demonstrate how different configuration and coding practices affect temporary file accumulation and disk usage.

> [!IMPORTANT]
> Overview about how Azure Function Apps operate within App Service infrastructure, focusing on temp file creation, storage, and management. Runtime behavior, deployment impact, and optimization strategies. For official guidance, support, or more detailed information, please refer to Microsoft's official documentation or contact Microsoft directly: [Microsoft Sales and Support](https://support.microsoft.com/contactus?ContactUsExperienceEntryPointAssetId=S.HP.SMC-HOME)

<details>
<summary><b>List of References</b> (Click to expand)</summary>
  
- [Kudu service overview](https://learn.microsoft.com/en-us/azure/app-service/resources-kudu)
- [log levels types](https://learn.microsoft.com/en-us/azure/azure-functions/configure-monitoring?tabs=v2#configure-log-levels)
- [How to configure monitoring for Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/configure-monitoring?tabs=v2)
- [host.json reference for Azure Functions 2.x and later](https://learn.microsoft.com/en-us/azure/azure-functions/functions-host-json#override-hostjson-values)
- [Sampling overrides %](https://learn.microsoft.com/en-us/azure/azure-monitor/app/java-standalone-config#sampling-overrides)
- [Sampling in Azure Monitor Application Insights with OpenTelemetry](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-sampling)
- [Azure Functions deployment technologies](https://learn.microsoft.com/en-us/azure/azure-functions/functions-deployment-technologies)
- [Run your Azure Functions from a package file](https://learn.microsoft.com/en-us/azure/azure-functions/run-functions-from-deployment-package)
- [Continuous delivery by using Azure DevOps](https://learn.microsoft.com/en-us/azure/azure-functions/functions-continuous-deployment)
- [Continuous delivery by using GitHub Actions](https://learn.microsoft.com/en-us/azure/azure-functions/functions-how-to-github-actions)
- [Best practices for reliable Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/functions-best-practices)
- [Improve the performance and reliability of Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/performance-reliability)
- [Accessing the kudu service](https://github.com/projectkudu/kudu/wiki/Accessing-the-kudu-service) - GitHub repo
- [Understanding the Azure App Service file system](https://github.com/projectkudu/kudu/wiki/Understanding-the-Azure-App-Service-file-system#temporary-files) - GitHub repo

</details>

<details>
<summary><b>Table of Content</b> (Click to expand)</summary>

- [Scenarios](#scenarios)
- [How to Compare Results](#how-to-compare-results)
- [Deployment Approaches](#deployment-approaches)
    - [High-Decay ](#high-decay-writable-approach) - `Writable Approach + Configs`
    - [Optimized](#optimized-mounted-package-approach) - `Mounted Package Approach + Configs`

</details>

> [!NOTE]
> - Azure Function App: This is the main service where your code lives and runs. Think of it as the container for your serverless functions.
>   - You write small pieces of code called functions that respond to events, like HTTP requests, database changes, or scheduled timers.
>   - These functions are event-driven or scheduled, meaning they only run when triggered, saving resources and cost.
> - App Service Runtime: This layer is the execution environment that powers your Function App.
>   - Language Runtimes: Supports multiple languages like C#, JavaScript, Python, etc.
>   - HTTP Framework / Middleware: Handles incoming HTTP requests and routes them to the correct function.
>   - Application Insights: Monitors and logs telemetry data for performance and debugging.
>   - Kudu / IIS: Manages deployment, diagnostics, and request/response handling.
>   - Hosts Function App: This is the actual runtime that executes your function code.
> - App Service Plan: This is the infrastructure layer that supports the runtime.
>   - Operating System: Can be Windows or Linux, depending on your configuration.
>   - Temporary Storage: Uses `C:\Local\Temp\` for non-persistent file storage during function execution.

<div align="center">
  <img src="https://github.com/user-attachments/assets/4964fb4e-360c-4844-afda-7d0cddc1593c" alt="Centered Image" style="border: 2px solid #4CAF50; border-radius: 5px; padding: 5px;"/>
</div>

> [!TIP]
> When a trigger (like an HTTP request or a timer) fires: 
> - The Function App receives the event.
> - The App Service Runtime processes it using the appropriate language runtime and middleware.
> - The function executes, possibly logging data to Application Insights.
> - The underlying OS and storage support the execution environment.

`This setup allows developers to focus purely on writing code without worrying about servers, scaling, or infrastructure management, making Azure Functions a powerful serverless computing option.`

## Scenarios

1. [High Decay test](./scenario1-high-decay): Test rapid temp file accumulation and disk decay
2. Recommendations for [Optimized configuration](./scenario2-optimized): Test how to minimize temp file accumulation
   
## How to Compare Results

> When deployed, two scenarios were compared by:

1. Monitoring disk usage in Kudu (`https://<function-app-name>.scm.azurewebsites.net/DebugConsole`)
2. Running load tests against both function apps
3. Observing memory usage and response times
4. Checking temp directories for file accumulation

## Deployment Approaches

Here are a few deployment examples that show different ways you can approach:

1. [VS manual deployment](./_deployment-options/VS-deployment-extension-manual-approach.md) - VS extension manual approach
2. [Writable Approach](./_deployment-options/az-functionapps-cicd-pipeline-template-writable.yml) - Deployment pipeline ADO approach
3. [Mounted Package Approach](./_deployment-options/az-functionapps-cicd-pipeline-template-optimized.yml) - Deployment pipeline ADO approach

###  High-Decay (Writable Approach)

> The combination of writable deployment, intensive logging, and full diagnostics causes Azure Functions to generate and buffer a large amount of telemetry and log data in local temp storage. This leads to rapid disk usage growth, temp file accumulation, and eventual disk decay. Click here to go to [High-Decay](./scenario1-high-decay) test approach

- **Deployment Method**: Standard deployment (extracted to wwwroot)
- **File Access**: Files are writable by the Function App
- **Pipelines**: Azure DevOps pipeline with standard deployment

> [!IMPORTANT]
> Overall, the problem with standard/writable deployment with intensive logging:
> - **Intensive logging and diagnostics** (full Application Insights, detailed diagnostics, verbose log level) generate a large volume of log and telemetry data.
> - **Writable deployment** (`WEBSITE_RUN_FROM_PACKAGE = 0`) means the function app runs from extracted files in `wwwroot`, allowing the app and platform to write files locally.
> - **Azure Functions and the platform buffer logs, telemetry, and temp files in the local file system**, specifically under `C:\local\Temp` and `D:\local\Temp` (on Windows plans).
> - **Telemetry and diagnostic logs are first written to local temp storage** before being sent to Application Insights or other destinations.
> - **High load and verbose logging** cause rapid accumulation of files in these temp directories.
> - **Disk usage grows over time** as temp files, logs, and telemetry buffers accumulate, especially if the app is under sustained or bursty load.
> - **Some files may remain locked or not be cleaned up automatically**, even after a function app restart, leading to "disk decay" (progressive loss of available disk space).
> - **Performance degrades** as disk fills up, and the app may eventually fail if the temp storage is exhausted.

### Optimized (Mounted Package Approach):

> The combination of mounted (read-only) deployment, optimized logging, and minimal diagnostics causes Azure Functions to generate and buffer very little telemetry and log data in local temp storage. This prevents disk usage from growing, avoids temp file accumulation, and eliminates disk decay. Click here to go to [Optimized](./scenario2-optimized) test approach

- **Deployment Method**: ZipDeploy with `WEBSITE_RUN_FROM_PACKAGE=1`
- **File Access**: Files are read-only (mounted from zip)
- **Pipelines**: Azure DevOps pipeline with ZipDeploy or GitHub Actions workflow

> [!IMPORTANT]
> **Overall, the benefit with optimized (mounted) deployment and minimal logging:**
> - **Minimal logging and diagnostics** (Application Insights with sampling, reduced diagnostics, less verbose log level) generate far less log and telemetry data.
> - **Mounted deployment** (`WEBSITE_RUN_FROM_PACKAGE = 1`) means the function app runs from a read-only mounted package, preventing the app and platform from writing files to `wwwroot`.
> - **Azure Functions and the platform buffer far fewer logs and temp files in the local file system**, reducing writes to `C:\local\Temp` and `D:\local\Temp`.
> - **Telemetry and diagnostic logs are sampled and sent directly to Application Insights or other destinations,** with minimal local buffering.
> - **Even under high load, temp file accumulation is negligible** because the app and platform cannot write to the mounted package and logging is optimized.
> - **Disk usage remains stable over time** as temp files, logs, and telemetry buffers are minimized and cleaned up efficiently.
> - **Restarts are rarely needed** for disk cleanup, and locked files are uncommon.
> - **Performance remains consistent** even under sustained load, as disk space is preserved and temp file decay is prevented.


