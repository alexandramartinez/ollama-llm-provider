# Salesforce's LLM Open Connector + MuleSoft (MAC) + Ollama

Mule application implementing Salesforce's LLM Open Connector using the MAC project. Our Ollama host is running locally for quick testing and we are using ngrok to expose Ollama's URL.

- [salesforce/einstein-platform](https://github.com/salesforce/einstein-platform)
- [Einstein Platform Cookbook](https://opensource.salesforce.com/einstein-platform/)
- [The MuleSoft AI Chain project](https://mac-project.ai/)

## Similar repos

[![](https://github-readme-stats.vercel.app/api/pin/?username=ProstDev&repo=mac-ollama-proj&theme=graywhite)](https://github.com/ProstDev/mac-ollama-proj)

## Steps

### Prerequisites

- Anypoint Platform account (create a free trial [here](https://anypoint.mulesoft.com/login/signup))

- [Anypoint Code Builder](https://docs.mulesoft.com/anypoint-code-builder/start-acb-desktop)

- [Ollama](https://ollama.com/) running locally + the model of your choice (i.e., `llama3`)

- [ngrok](https://download.ngrok.com/) to expose your local Ollama instance to the public cloud

- Any REST client application to test the API calls. For example, cURL or Postman.

### 1 - Install Ollama locally

- Go to [ollama.com](https://ollama.com/) and follow the prompts to install in your local computer

- Make a note of which version you started running (like `llama3` or `llama3.2`)

- To verify ollama is running, you should either be able to interact with it in the terminal or you should be able to run `ollama list` or `ollama ps`

- Run the following command to start ngrok:

    ```shell
    ngrok http 11434 --host-header="localhost:11434"
    ```

- Copy the address from the **Forwarding** field. This is the URL you will use to connect to Ollama from the Mule app.

### 2 - Create and publish the API specification

- Download the `llm-open-connector.yml` file from the [einstein-platform](https://github.com/salesforce/einstein-platform/blob/main/api-specs/llm-open-connector/llm-open-connector.yml) GitHub repository.

- Rename the extension of the file from `yml` to `yaml`.

- Head to your [Anypoint Platform](https://anypoint.mulesoft.com) account.

- Navigate to **Design Center**.

- Click on **Create +** > **Import from File**.

- Fill out the following details:

    - Project name: `LLM Open Connector`
    - Import type: API specification `REST API`
    - Import file: the file you downloaded on step 1

- Click on **Import**

- Add a character inside `termsOfService` (i.e., `termsOfService: "-"`)

- Remove the `servers` section:

    ```yaml
    servers:
        - url: https://bring-your-own-llm.example.com
    ```

- Click on **Publish** at the top-right side of the screen.

- Make sure the following details are selected:

    - Asset version: `1.0.0`
    - API version: `v1`
    - LifeCycle State: Stable

- Click on **Publish to Exchange**.

- Once published, you can click on the **Exchange** link to see the preview of the published asset, or click on **Close**.

- Exit Design Center

### 3 - Implement the Mule app
- In ACB, click on **Implement an API**.

> [!NOTE]
> If you haven't signed in to your Anypoint Platform account through ACB, it will ask you to sign in now. Follow the prompts to sign in.

- Fill out the following details:

    - Project Name: `ollama-llm-provider`
    - Project Location: choose any folder to keep this project
    - Search an API Specification from Exchange to implement it: the **LLM Open Connector** we just published
    - Mule Runtime: `4.8.0`
    - Java Version: `17`

- Click on **Create Project** and wait for it to be fully processed.

#### Maven

- Once processed, open the `pom.xml` file and add the following Maven dependency:

    ```xml
    <dependency>
        <groupId>com.mulesoft.connectors</groupId>
        <artifactId>mule4-aichain-connector</artifactId>
        <version>1.0.0</version>
        <classifier>mule-plugin</classifier>
    </dependency>
    ```

- Change the `version` (should be line 6) to the following:
    
    ```xml
    <version>1.0.0</version>
    ```
 
- Copy the `groupId` (in `dependencies`) for the `llm-open-connector` artifact. If should be a number similar to this: `d62b8a81-f143-4534-bb89-3673ad61ah01`. This is your organization ID.

- Paste this organization ID in the `groupId` field at the top of the file (should be line 4), replacing `com.mycompany` with your org ID. It should look similar to this:

    ```xml
    <groupId>d62b8a81-f143-4534-bb89-3673ad61ah01</groupId>
    ```

#### LLM Config

- Create a new file under src/main/resources named `llm-config.json`.

- Paste the following content into this file:

    ```json
    {
        "OLLAMA": {
            "OLLAMA_BASE_URL": "https://11-13-23-16-11.ngrok-free.app"
        }
    }
    ```

- Replace the example URL with the one you copied from ngrok.

#### Mule Flow

- Open the `ollama-llm-provider.xml` file under src/main/mule.

- Add the following code under the `apikit:config` and before the `<flow name="llm-open-connector-main">` (should be line 7)

    ```xml
    <ms-aichain:config configType="Configuration Json" filePath='#[(mule.home default "") ++ "/apps/" ++ (app.name default "") ++ "/llm-config.json"]' llmType="OLLAMA" modelName="#[vars.inputPayload.model]" name="MAC_Config" temperature="#[vars.inputPayload.temperature]" maxTokens="#[vars.inputPayload.max_tokens]"></ms-aichain:config>
    ```

- Locate the last flow in the file (should be `post:\chat\completions:application\json:llm-open-connector-config`).

- Remove the logger and add the following code inside this flow.

    ```xml
    <logger doc:name="Logger" doc:id="ezzhif" message="#[payload]" />
    <ee:transform doc:name="Transform" doc:id="dstcls">
        <ee:message>
            <ee:set-payload><![CDATA[payload.messages[0].content]]></ee:set-payload>
        </ee:message>
        <ee:variables>
            <ee:set-variable variableName="inputPayload">
                <![CDATA[payload]]>
            </ee:set-variable>
        </ee:variables>
    </ee:transform>
    <ms-aichain:chat-answer-prompt config-ref="MAC_Config" doc:id="mmoptd" doc:name="Chat answer prompt" />
    <ee:transform doc:name="Transform" doc:id="czdqgi">
        <ee:message>
            <ee:set-payload>
                    <![CDATA[output application/json
    ---
    {
        id: uuid( ),
        choices: [
            {
                finish_reason: "stop",
                index: 0,
                message: {
                    content: payload.response,
                    role: vars.inputPayload.messages[0].role
                }
            }
        ],
        created: now() as Number,
        model: vars.inputPayload.model,
        object: "chat.completion",
        usage: {
            completion_tokens: (attributes.tokenUsage.outputCount default 0) as Number,
            prompt_tokens: (attributes.tokenUsage.inputCount default 0) as Number,
            total_tokens: (attributes.tokenUsage.totalCount default 0) as Number
        }
    }]]>
            </ee:set-payload>
        </ee:message>
    </ee:transform>
    <logger doc:name="Logger" doc:id="rzhuiw" message="#[payload]" />
    ```

### 4 - Deploy to CloudHub 2.0

> [!NOTE]
> You can test your application locally before deploying to CloudHub to verify everything was set up correctly. To do this, head to the **Run and Debug** tab, press the green run button, and use the [Test the deployed Mule app](#5---test-the-deployed-mule-app) instructions. Just replace the URL with `http://localhost:8081/api/chat/completions`

- Still with the `ollama-llm-provider.xml` file open, click on the **Deploy to CloudHub** button located at the top-right side of the screen (the little rocket 🚀 icon).

- Select **CloudHub 2.0**.

- Select **Cloudhub-US-East-2** if you have a free trial account based on the US cloud. Otherwise, select any of the regions available to you.
    
> [!NOTE]
> To double-check the regions you have access to in your Anypoint Platform account, you can do so from **Runtime Manager** by trying to create a new deployment and seeing the available options.
    
- Select **Sandbox** as the environment.

- A new file with the name `deploy_ch2.json` will be created under src/main/resources. You can change the data in this file if needed. If not, it should look like this:

    ```json
    {
        "applicationName": "ollama-llm-provider",
        "runtime": "4.8.0",
        "replicas": 1,
        "replicaSize": "0.1",
        "deploymentModel": "rolling"
    }
    ```

- Click on **Deploy** in the message that appears at the bottom-right of the screen.

- Select the latest Mule runtime version available from the next prompt. In this case, you can select `4.8.1:6e-java17` if available. This will change the `runtime` field from the `deploy_ch2.json` file.

- Your asset will first be published to Exchange as an Application. After that, the deployment to CloudHub 2.0 will start. Once you receive the *'ollama-llm-provider' is deployed to 'Sandbox' Env in CloudHub 2.0*, it means the deployment has successfully started in Anypoint Platform.

- Head to your Anypoint Platform account and open **Runtime Manager** (or click the **Open in Runtime Manager** button from ACB).

- Once your application's status appears as 🟢 `Running`, it means it's ready to start receiving requests. Copy the URL (Public Endpoint). It should look similar to this: `https://ollama-llm-provider-69s5jr.5se6i9-2.usa-e2.cloudhub.io`.

- Add `/api` at the end of this URL. This is the URL we will use to call this API.

### 5 - Test the deployed Mule app

You can call the previous URL using tools like cURL or Postman. Here is an example cURL command you can use, just make sure you replace the example URL with your own.

```shell
curl -X POST https://ollama-llm-provider-69s5jr.5se6i9-2.usa-e2.cloudhub.io/api/chat/completions \
-H 'Content-Type: application/json' \
-d '{
  "messages": [
    {
      "content": "What is the capital of Canada?",
      "role": "user",
      "name": "Alex"
    }
  ],
  "model": "llama3",
  "max_tokens": 0,
  "n": 1,
  "temperature": 1,
  "parameters": {
    "top_p": 0.5
  }
}'
```

> [!NOTE]
> Since Ollama does not require an `api-key` header, we are not sending it in this request.
