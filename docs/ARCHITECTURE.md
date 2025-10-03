# API Relay Architecture

## 1. Introduction

This document provides a comprehensive overview of the API relay system's architecture. The system is designed as a sophisticated intermediary that provides a unified interface for various downstream AI services, including Anthropic's Claude, Google's Gemini, and OpenAI's models (as used by Codex).

The primary goal of this architecture is to offer a consistent and managed access layer, abstracting the complexities of interacting with different API providers. It achieves this through a modular design that includes intelligent routing, request/response translation, and a robust account management system.

## 2. Core Components

The system is built upon several key components that work together to process API requests:

-   **Express.js Server**: The foundation of the application, handling all incoming HTTP requests and routing them to the appropriate controllers.
-   **API Routers**: A set of specialized routers, each responsible for a specific API endpoint or compatibility layer (e.g., `/api`, `/openai`, `/gemini`).
-   **Authentication Middleware**: Ensures that all incoming requests have a valid API key and that the key has the necessary permissions for the requested service.
-   **Translation Services**: A crucial layer that converts requests and responses between the standard OpenAI format and the native formats of Claude and Gemini. This allows clients to use a single, consistent API format to interact with multiple backend services.
-   **Unified Schedulers**: Intelligent services responsible for selecting the most appropriate backend account for a given request. This includes logic for handling account pools, dedicated account bindings, session stickiness, and rate-limit avoidance.
-   **Relay Services**: These services are responsible for the final step of forwarding the processed request to the actual upstream API (Claude, Gemini, or OpenAI).

## 3. Request Flow

A typical API request to the relay follows these steps:

1.  **Request Reception**: The Express.js server receives an incoming API request.
2.  **Authentication**: The `authenticateApiKey` middleware validates the API key provided in the request headers. It checks for the key's existence, validity, and permissions.
3.  **Routing**: The request is passed to the appropriate router based on its URL path (e.g., a request to `/openai/v1/chat/completions` is handled by the OpenAI compatibility router).
4.  **Request Translation (if necessary)**: If the request is to a compatibility endpoint (like the OpenAI-to-Claude or OpenAI-to-Gemini routes), the request body is converted from the OpenAI format to the target service's native format.
5.  **Account Selection**: The unified scheduler for the target service (e.g., `unifiedClaudeScheduler`) is invoked. It selects the best account based on:
    *   API key bindings (dedicated accounts).
    *   Session stickiness (to maintain context with the same backend account).
    *   Account availability, priority, and rate-limit status.
    *   Model compatibility.
6.  **Request Relaying**: The final, translated request is forwarded to the selected upstream API provider.
7.  **Response Handling**: The response from the upstream API is received.
8.  **Response Translation (if necessary)**: If the request was made to a compatibility endpoint, the response is converted back to the standard OpenAI format.
9.  **Usage Recording**: The token usage from the response is recorded and associated with the API key.
10. **Response to Client**: The final, formatted response is sent back to the original client.

## 4. Multi-API Compatibility

A core feature of this architecture is its ability to seamlessly interact with multiple AI providers through a unified interface. This is primarily achieved through the compatibility routers:

-   **`openaiClaudeRoutes.js`**: This router exposes an endpoint that mimics the OpenAI `chat/completions` API but directs the requests to the Claude API. It uses the `openaiToClaude.js` service to translate both the outgoing requests and the incoming responses, allowing any OpenAI-compatible client to use Claude models.
-   **`openaiGeminiRoutes.js`**: Similarly, this router provides an OpenAI-compatible endpoint for the Gemini API. It handles the translation between the two formats, enabling OpenAI clients to leverage Gemini models.
-   **`openaiRoutes.js`**: This router is specifically designed to be compatible with the Codex CLI, which uses a variant of the OpenAI API.

This design allows users to switch between different foundation models without changing their client-side code, simply by targeting a different endpoint on the relay.

## 5. Account Management and Scheduling

The account management system is a cornerstone of the relay's reliability and efficiency.

-   **Unified Schedulers**: The `unifiedClaudeScheduler.js` and `unifiedOpenAIScheduler.js` are central to this system. They provide a single point of logic for selecting an account from a pool of available options.
-   **Account Pooling**: The system can manage a shared pool of accounts for each service. The schedulers rotate through these accounts based on priority and last-used time to distribute the load.
-   **Session Stickiness**: To ensure conversational context is maintained, the schedulers can map a user's session to a specific account for a configurable duration. This means subsequent requests from the same user will be sent to the same backend account.
-   **Rate-Limit and Error Handling**: The schedulers are designed to be resilient. They can detect when an account is rate-limited or has encountered an error, automatically taking it out of rotation and selecting the next available account. This prevents a single failing account from disrupting the entire service.
-   **Dedicated Accounts**: API keys can be bound to specific accounts or groups of accounts, providing a way to guarantee resources for particular users or applications.

## 6. Conclusion

The API relay architecture is a powerful and flexible system for managing access to multiple AI services. Its modular design, combined with intelligent routing, API translation, and robust account scheduling, provides a reliable and unified interface that simplifies the use of diverse AI models. This architecture is well-suited for environments where managing costs, ensuring reliability, and providing a consistent developer experience are top priorities.