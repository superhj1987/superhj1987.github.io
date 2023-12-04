---
layout: post
title: "Langchain代理和OpenAI函数调用的区别"
date: 2023-11-30 19:29:34 +0800
comments: true
categories: GenAI
---

最近在实现LLM调用外部工具的时候，突然意识到貌似OpenAI的Function Calling和LangChain的Agent都能达到相同的结果，只是实现方式不同。因此这里来对比一下。

<!--more-->

## OpenAI Functions Calling

OpenAI函数允许更好地控制AI调用的函数，因为我们必须解析AI的响应，找出AI想要调用的函数，并将参数传递给该函数。我们可以在每个阶段干预并修改被调用函数或其参数。

假设我们想让AI使用一个REST API，而不指定它可以执行的操作。我们提供一个通用的REST API客户端，让AI决定使用哪种HTTP方法和哪些参数。

## 定义一个函数

首先，我们必须准备一个可用Function的描述。

```
functions = [
{
    "type": "function",
    "function": {
        "name": "call_rest_api",
        "description": "Sends a request to the REST API",
        "parameters": {
            "type": "object",
            "properties": {
                "method": {
                    "type": "string",
                    "description": "The HTTP method to be used",
                    "enum": ["GET", "POST", "PUT", "DELETE"],
                },
                "url": {
                    "type": "string",
                    "description": "The URL of the endpoint. Value placeholders must be replaced with actual values.",
                },
                "body": {
                    "type": "string",
                    "description": "A string representation of the JSON that should be sent as the request body.",
                },

            },
            "required": ["method", "url"],
        },
    }
}
]
```

除了描述之外，还需要一个函数的实现。毕竟，当我们收到一个表示AI想要调用函数的响应时，我们将负责调用该函数。

```
def call_rest_api(self, arguments):
    arguments = json.loads(arguments)
    # reques.in is a hosted, fake REST API that we can use for testing
    url = 'https://reqres.in' + arguments['url']
    body = arguments.get('body', {})
    response = None
    if arguments['method'] == 'GET':
        response = requests.get(url)
    elif arguments['method'] == 'POST':
        response = requests.post(url, json=body)
    elif arguments['method'] == 'PUT':
        response = requests.put(url, json=body)
    elif arguments['method'] == 'DELETE':
        response = requests.delete(url)
    else:
        raise ValueError(arguments)

    if response.status_code == 200:
        return json.dumps(response.json())
    else:
        return f"Status code: {response.status_code}"
```

该函数是通用的，可以处理发送到任何端点的任何HTTP请求。需要列出可用的端点和支持的HTTP方法。我更喜欢将这些参数收集到一个变量中，稍后用于生成提示：

```
available_apis = [
        {'method': 'GET', 'url': '/api/users?page=[page_id]', 'description': 'Lists employees. The response is paginated. You may need to request more than one to get them all. For example,/api/users?page=2.'},
        {'method': 'GET', 'url': '/api/users/[user_id]', 'description': 'Returns information about the employee identified by the given id. For example,/api/users/2'},
        {'method': 'POST', 'url': '/api/users', 'description': 'Creates a new employee profile. This function accepts JSON body containing two fields: name and job'},
        {'method': 'PUT', 'url': '/api/users/[user_id]', 'description': 'Updates employee information. This function accepts JSON body containing two fields: name and job. The user_id in the URL must be a valid identifier of an existing employee.'},
        {'method': 'DELETE', 'url': '/api/users/[user_id]', 'description': 'Removes the employee identified by the given id. Before you call this function, find the employee information and make sure the id is correct. Do NOT call this function if you didn\'t retrieve user info. Iterate over all pages until you find it or make sure it doesn\'t exist'}
    ]
```

接着需要定义AI的任务和可用的功能。为此，准备以下系统提示词。

```
messages = [
    {"role": "system", 
    "content": "You are an HR helper who makes API calls on behalf of an HR representative，You have access to the following APIs: " 
        + json.dumps(self.available_apis)
        + "If a function requires an identifier, list all employees first to find the proper value. You may need to list more than one page"
        + "If you were asked to create, update, or delete a user, perform the action and reply with a confirmation telling what you have done"
    }
]
```

最终，可以与AI进行交互。我们定义一个函数call_ai。该函数接受用户的提示，并将提示传递给之前定义的OpenAI模型的上下文描述。上下文描述还包含了可用函数的描述。如果AI回复的消息包含function_call，我们会调用一个函数。该消息包括函数名和参数。

```
def call_ai(self, new_message):
    if new_message:
        self.messages.append({"role": "user", "content": new_message})

    response = self.client.chat.completions.create(
        model="gpt-3.5-turbo-1106",
        messages=self.messages,
        tools=self.functions,
        tool_choice="auto",
    )

    msg = response.choices[0].message
    self.messages.append(msg)
    if msg.content:
        logging.debug(msg.content)
    if tool_calls:
        msg.content = "" # required due to a bug in the SDK. We cannot send a message with None content
        for tool_call in tool_calls:
            # we may get a request to call more than one function(!)
            function_name = tool_call.function.name
            function_args = json.loads(tool_call.function.arguments)
            if function_name == 'call_rest_api':
                # ['function_call']['arguments'] contains the arguments of the function
                logging.debug(function_args)
                # Here, we call the requested function and get a response as a string
                function_response = self.call_rest_api(function_args)
                logging.debug(function_response)
                # We add the response to messages
                self.messages.append({
                    "tool_call_id": tool_call.id,
                    "role": "tool",
                    "name": function_name,
                    "content": function_response
                })

                self.call_ai(new_message=None) # pass the function response back to AI
    else:
      # return the final response
      return msg
```

一方面，这样的实现使代码变得非常冗长。就像下面所看到的，我们必须选择要调用的Python函数，传递参数，将响应添加到聊天记录中（作为一个带有role设置为tool的消息），然后再次调用AI并更新聊天记录。

另一方面，我们对被调用的函数保持完全控制。我们可以调用不同的函数，改变参数，或者传递一个虚假的响应给人工智能，而不是调用一个函数。

通过OpenAI的实现，我们还可以轻松创建长时间运行的工作流程，在数据库中存储聊天记录，在AI请求的函数调用后等待几个小时或几天，当我们收到回复时继续工作流程。

## 使用Langchain代理进行函数调用

Langchain代理隐藏了调用函数的复杂性。我们仍然需要提供函数描述和实现，或者使用其中一个可用的Langchain工具包。Langchain工具包是一种快捷方式，允许我们跳过编写函数及其描述的步骤。相反，我们使用工具包作者准备的代码。

在Langchain中，我们必须选择其中一种可用的代理类型。代理类型定义了提示，以便向模型介绍可用的功能。

代理类型实现了各种交互样式。例如，在conversational-react-description代理中，提示的指令部分如下所示：

```
To use a tool, please use the following format:

Thought: Do I need to use a tool? Yes
Action: the action to take, should be one of [{tool_names}]
Action Input: the input to the action
Observation: the result of the action

When you have a response to say to the Human, or if you do not need to use a tool, you MUST use the format:

Thought: Do I need to use a tool? No
{ai_prefix}: [your response here]

## https://github.com/langchain-ai/langchain/blob/9731ce5a406d5a7bb1878a54b265a6f7c728effc/libs/langchain/langchain/agents/conversational/prompt.py
```

当使用此提示时，Langchain会让AI生成输出，直到它产生Action Input: 之后的第一个换行符。此时，Langchain会中断AI调用，找到使用Action: 行返回的名称的函数，并将Action Input: 的值作为参数传递。

稍后，当函数返回响应时，响应被添加到提示中的Observation: 行，并且Langchain再次调用LLM来继续处理提示。LLM可能会产生另一个Action: + Action Input: 对，以调用另一个函数或生成以Final Answer: 开头的行，将响应返回给用户。

一个流行的zero-shot-react-description几乎具有相同的提示。

## 在Langchain中使用OpenAI Function Calling

选择一个代理类型，该类型使用OpenAI函数，但隐藏了选择函数和传递参数的复杂性。使用配置Langchain代理的标准代码结构，但选择OPENAI_FUNCTIONS AgentType。

Langchain试图将整个函数描述打包成一条消息，而长描述很容易超过消息长度限制，因此不得不缩短提示。如果需要所有的端点，可以将call_rest_api函数拆分成多个函数，每个函数处理一个端点。
```
import json
import requests
from langchain import OpenAI
from langchain.agents import initialize_agent, Tool
from langchain.agents import AgentType
from langchain.chat_models import ChatOpenAI


def call_rest_api(method_and_url):
    method, url = method_and_url.split(' ')
    url = 'https://reqres.in' + url
    response = None
    if method == 'GET':
        response = requests.get(url)
    elif method == 'DELETE':
        response = requests.delete(url)
    else:
        raise ValueError(method)

    if response.status_code == 200:
        return response.json()
    else:
        return f"Status code: {response.status_code}"

available_apis = [
        {'api ': 'GET /api/users?page=[page_id]', 'description': 'Lists employees. The response is paginated. You may need to request more than one to get them all. For example,/api/users?page=2.'},
        {'api': 'DELETE /api/users/[numeric user_id]', 'description': 'Removes the employee identified by the given id. Before you call this function, find the employee information and make sure the id is correct. Do NOT call this function if you didn\'t retrieve user info. Iterate over all pages until you find it or make sure it doesn\'t exist'}
    ]

api_description = json.dumps(available_apis)

function_description = f"""Sends a request to the REST API.

Accepts a single parameter containing two parts:
* method - The HTTP method to be used (GET, POST, PUT, or DELETE)
* url - The URL of the endpoint. Value placeholders must be replaced with actual values.

For example: GET /api/users?page=1 or DELETE /api/users/13

To find users by name, use the GET method first.

Available endpoints:
{api_description}"""

llm = ChatOpenAI(temperature=0, model="gpt-4", openai_api_key='sk-...')
tools = [
    Tool(
        name = "REST",
        func=call_rest_api,
        description=function_description
    )
]

agent = initialize_agent(tools, llm, agent=AgentType.OPENAI_FUNCTIONS, verbose=True)
agent.run("Fire Lawson")
```

在控制台中，可以看到Agent的详细日志：

```
Invoking: `REST` with `GET /api/users?page=1`
{'page': 1, 'per_page': 6, 'total': 12, 'total_pages': 2, 'data': [...

Invoking: `REST` with `GET /api/users?page=2`
{'page': 2, 'per_page': 6, 'total': 12, 'total_pages': 2, 'data': [{'id': 7, 'email': 'michael.lawson@reqres.in', 'first_name': 'Michael', 'last_name': 'Lawson', 'avatar': 'https://reqres.in/img/faces/7-image.jpg'}, ...

Invoking: `REST` with `DELETE /api/users/7`
Status code: 204

User Lawson has been successfully removed from the system.
```

## 如何选择？

如果我有一个简单的函数和简短的描述，优先选择Langchain，因为Langchain隐藏了许多实现细节，让我们专注于准备提示。

当函数的描述不再适合于一条信息时，那只能使用OpenAI函数，而不使用Langchain。

此外，调试提示词时，建议切换到OpenAI函数。它不会神奇地解决问题，必须编写整个函数调用代码。因此使得调试将更加容易。