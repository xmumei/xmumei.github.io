---
layout: mypost
title: 使用Groq免费调用DeepSeekAPI并通过Deno实现墙内访问
categories: [瞎折腾]
---

**DeepSeek 很强, 但 API 要钱. Groq 免费, 但墙内不能访问. 遂使用 Deno 转发.**

### 通过 Groq 获取 Secret Key

![1](1.png)

![2](2.png)

得到 Secret Key 后, 可以使用下面这段 curl 脚本进行测试, 注意将 Authorization 中的汉字替换为你得到的 API:

```bash
curl https://api.groq.com/openai/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer 将这里替换为你的API" \
  -d '{
    "model": "deepseek-r1-distill-llama-70b",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "你是什么模型？"}
    ]
  }'
```

请求后返回的结果应该是类似下面的 JSON 结构:

```json
{
  "id": "...",
  "object": "chat.completion",
  "created": ...,
  "model": "deepseek-r1-distill-llama-70b",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "我是 deepseek-r1-distill-llama-70b 模型..."
      },
      ...
    }
  ],
  ...
}
```

或者一串报错:

```json
{"error":{"message":"Not Found"}}%
```

出现报错的原因是使用国内网络, 因此需要使用 Deno 来作为国内访问代理.

### Deno 转发

Deno 是一个现代化的 Javascript 运行时, 内置 HTTP 服务器, 很适合快速搭建代理.

![3](3.png)

![4](4.png)

要粘贴的代码如下:

````js
interface RateLimiter {
  requests: number;
  tokens: number;
  lastReset: number;
}

const rateLimiter: RateLimiter = {
  requests: 0,
  tokens: 0,
  lastReset: Date.now(),
};

function estimateTokens(body: any): number {
  try {
    const messages = body?.messages || [];
    return messages.reduce((acc: number, msg: any) =>
      acc + (msg.content?.length || 0) * 0.25, 0);
  } catch {
    return 0;
  }
}

function resetCountersIfNeeded() {
  const now = Date.now();
  if (now - rateLimiter.lastReset >= 60000) {
    rateLimiter.requests = 0;
    rateLimiter.tokens = 0;
    rateLimiter.lastReset = now;
  }
}

async function processResponse(response: Response): Promise<Response> {
  const contentType = response.headers.get('content-type');
  if (contentType?.includes('application/json')) {
    const jsonData = await response.json();

    if (jsonData.choices && jsonData.choices[0]?.message?.content) {
      const content = jsonData.choices[0].message.content;
      const processedContent = content.replace(/<think>.*?<\/think>\s*/s, '').trim();
      jsonData.choices[0].message.content = processedContent;
    }

    return new Response(JSON.stringify(jsonData), {
      status: response.status,
      headers: response.headers
    });
  }

  return response;
}

async function handleRequest(request: Request): Promise<Response> {
  const url = new URL(request.url);
  const pathname = url.pathname;

  if (pathname === '/' || pathname === '/index.html') {
    return new Response('Proxy is Running！', {
      status: 200,
      headers: { 'Content-Type': 'text/html' }
    });
  }

  if (pathname.includes('api.groq.com')) {
    resetCountersIfNeeded();

    if (rateLimiter.requests >= 30) {
      return new Response('Rate limit exceeded. Max 30 requests per minute.', {
        status: 429,
        headers: {
          'Retry-After': '60',
          'Content-Type': 'application/json'
        }
      });
    }

    try {
      const bodyClone = request.clone();
      const body = await bodyClone.json();
      const estimatedTokens = estimateTokens(body);

      if (rateLimiter.tokens + estimatedTokens > 6000) {
        return new Response('Token limit exceeded. Max 6000 tokens per minute.', {
          status: 429,
          headers: {
            'Retry-After': '60',
            'Content-Type': 'application/json'
          }
        });
      }

      rateLimiter.tokens += estimatedTokens;
    } catch (error) {
      console.error('Error parsing request body:', error);
    }

    rateLimiter.requests++;
  }

  const targetUrl = `https://${pathname}`;

  try {
    const headers = new Headers();
    const allowedHeaders = ['accept', 'content-type', 'authorization'];
    for (const [key, value] of request.headers.entries()) {
      if (allowedHeaders.includes(key.toLowerCase())) {
        headers.set(key, value);
      }
    }

    const response = await fetch(targetUrl, {
      method: request.method,
      headers: headers,
      body: request.body
    });

    const responseHeaders = new Headers(response.headers);
    responseHeaders.set('Referrer-Policy', 'no-referrer');
    responseHeaders.set('X-RateLimit-Remaining', `${30 - rateLimiter.requests}`);
    responseHeaders.set('X-TokenLimit-Remaining', `${6000 - rateLimiter.tokens}`);

    const processedResponse = await processResponse(response);

    return new Response(processedResponse.body, {
      status: processedResponse.status,
      headers: responseHeaders
    });

  } catch (error) {
    console.error('Failed to fetch:', error);
    return new Response(JSON.stringify({
      error: 'Internal Server Error',
      message: error.message
    }), {
      status: 500,
      headers: {
        'Content-Type': 'application/json'
      }
    });
  }
}

Deno.serve(handleRequest);
```

最后再用 curl 测试一下,

```bash
curl 将这里替换为上面生成的地址/api.groq.com/openai/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer 将这里替换为你的API" \
  -d '{
    "model": "deepseek-r1-distill-llama-70b",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "你是什么模型？"}
    ]
  }'
````

即使使用墙内网络也可以正常得到结果
