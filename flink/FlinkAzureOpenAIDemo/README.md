# Using Azure OpenAI with Apache Flink 

This code demonstrates how to integrate Apache Flink® in Azure HDInsight on AKS with Azure OpenAI to process a stream of data in real-time. 
It’s a powerful combination for big data processing and AI. 

Apache Flink is a framework for big data processing and distributed computation. It can handle both batch and stream processing.
Azure OpenAI is a cloud-based AI service that provides various AI models, including the GPT-4 model used in this code.
OpenAI’s GPT-4 model is capable of understanding and generating human-like text, making it a powerful tool for building intelligent chatbots.
With Azure OpenAI, customers get the security capabilities of Microsoft Azure while running the same models as OpenAI. 

To process the chatbot’s responses in real-time. This is where Apache Flink comes in. 
This blog code integrates Flink with Azure OpenAI to create an intelligent chatbot that can answer questions about U.S. Presidents. 
The chatbot’s responses are processed in real-time using Flink, allowing it to provide immediate feedback.

## Requirements
• An Azure subscription <br>
• Apache Flink® 1.17 in Azure HDInsight on AKS <br>
https://learn.microsoft.com/en-us/azure/hdinsight-aks/flink/flink-overview <br>
• Azure OpenAI Service <br>
https://learn.microsoft.com/en-us/azure/ai-services/openai/overview  <br>
• Maven project development on Azure VM in the same Vnet <br>

## Set up testing environment

**Retrieve key and endpoint of Azure OpenAI** <br>

https://learn.microsoft.com/en-us/azure/ai-services/openai/chatgpt-quickstart?tabs=command-line%2Cpython-new&pivots=programming-language-java#set-up <br>
To successfully make a call against Azure OpenAI, you need an endpoint and a key. <br>
```
Variable name	 Value
ENDPOINT	     This value can be found in the Keys & Endpoint section when examining your resource from the Azure portal. Alternatively, you can find the value in the Azure OpenAI Studio > Playground > Code View. An example endpoint is: https://docs-test-001.openai.azure.com/.
API-KEY	       This value can be found in the Keys & Endpoint section when examining your resource from the Azure portal. You can use either KEY1 or KEY2.
```

Go to your resource in the Azure portal. The Keys & Endpoint section can be found in the Resource Management section. Copy your endpoint and access key as you'll need both for authenticating your API calls. You can use either KEY1 or KEY2. Always having two keys allows you to securely rotate and regenerate keys without causing a service disruption.

## Here's a breakdown of what each part of the code in java does:

• The **main** method sets up the Flink execution environment, creates an input data stream, sets up the processing stream, prints the output data, and then executes the job. <br>
• The **AsyncHttpRequestFunction** class implements the AsyncFunction interface, which allows for asynchronous I/O operations in Flink. This is useful for operations that are I/O-bound, such as making HTTP requests. <br>
• The **asyncInvoke** method is where the actual processing happens. It takes an input string and a ResultFuture object. The input string is a name of a U.S. president, and the ResultFuture is used to output the result of the processing. <br>
Inside the asyncInvoke method, an OpenAIClient is created using the provided Azure OpenAI key, endpoint, and model ID. This client is used to interact with the OpenAI API.
A list of ChatMessage objects is created. These messages simulate a conversation with the OpenAI API. The conversation starts with a system message, followed by a user message asking for help, an assistant message offering help, and finally a user message asking for the birthday and term of presidency of the input president.
The getChatCompletions method of the OpenAIClient is called with the model ID and the list of chat messages. This sends the conversation to the OpenAI API and gets a response.
The response from the OpenAI API is printed out. This includes the model ID, the creation time of the model, the content of the messages, and the usage statistics. 
If an exception occurs during the processing, it’s caught and its stack trace is printed.

## Java Code
``` java
package contoso.example;

import com.azure.ai.openai.OpenAIClient;
import com.azure.ai.openai.OpenAIClientBuilder;
import com.azure.ai.openai.models.*;
import com.azure.core.credential.AzureKeyCredential;
import org.apache.flink.streaming.api.datastream.AsyncDataStream;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.async.AsyncFunction;
import org.apache.flink.streaming.api.functions.async.ResultFuture;

import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.concurrent.TimeUnit;

public class test2 {
    public static void main(String[] args) throws Exception {
        // create Apache Flink execution environment
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment().setParallelism(1);

        // create an input stream
        DataStream<String> testStream = env.fromElements("Bill Smith","Joe Biden","Donald John Trump","William Jefferson Clinton","Franklin Delano Roosevelt","Abraham Lincoln","George Washington");

        // create a processing stream
        DataStream<String> resultDataStream = AsyncDataStream.unorderedWait(
                testStream, new AsyncHttpRequestFunction(), 60000, TimeUnit.MILLISECONDS, 100);

        // output data
        resultDataStream.print();

        // execute the job
        env.execute("Flink plus OpenAI");
    }

    public static class AsyncHttpRequestFunction implements AsyncFunction<String, String> {

        @Override
        public void asyncInvoke(String input, ResultFuture<String> resultFuture) {
            OpenAIClient client = null;
            try {
                // OpenAI API endpoint for text completion
                String azureOpenaiKey = System.getenv("AZURE_OPENAI_API_KEY");;
                String endpoint = System.getenv("AZURE_OPENAI_ENDPOINT");;
                String deploymentOrModelId = "gpt-4";

                client = new OpenAIClientBuilder()
                        .endpoint(endpoint)
                        .credential(new AzureKeyCredential(azureOpenaiKey))
                        .buildClient();

                List<ChatMessage> chatMessages = new ArrayList<>();
                chatMessages.add(new ChatMessage(ChatRole.SYSTEM, "Assistant is an intelligent chatbot designed to help users answer their tax related questions.\n" +
                        "Instructions: \n" +
                        "- Only answer questions related to President of the United States. \n" +
                        "- If you're unsure of an answer, you can say \"I don't know\" or \"I'm not sure\" and recommend users go to the IRS website for more information. \""));
                chatMessages.add(new ChatMessage(ChatRole.USER, "Can you help me?"));
                chatMessages.add(new ChatMessage(ChatRole.ASSISTANT, "Of course! What can I do for you?"));
                chatMessages.add(new ChatMessage(ChatRole.USER, "Birthday and Which term of presidency of President of the United States: " + input));

                ChatCompletions chatCompletions = client.getChatCompletions(deploymentOrModelId, new ChatCompletionsOptions(chatMessages));

                System.out.printf("Model ID=%s is created at %s.%n", chatCompletions.getId(), chatCompletions.getCreatedAt());

                for (ChatChoice choice : chatCompletions.getChoices()) {
                    ChatMessage message = choice.getMessage();
                    System.out.printf("Index: %d, Chat Role: %s.%n", choice.getIndex(), message.getRole());
                    System.out.println("Message:");
                    System.out.println(message.getContent());
                }

                System.out.println();
                CompletionsUsage usage = chatCompletions.getUsage();
                System.out.printf("Usage: number of prompt token is %d, "
                                + "number of completion token is %d, and number of total tokens in request and response is %d.%n",
                        usage.getPromptTokens(), usage.getCompletionTokens(), usage.getTotalTokens());
            } catch (Exception e) {
                e.printStackTrace();
            }
            }

        @Override
        public void timeout(String input, ResultFuture<String> resultFuture) throws Exception {
            System.out.println("Timeout occurred for input: " + input);
            resultFuture.complete(Collections.emptyList());
        }

        }
    }
``` java

