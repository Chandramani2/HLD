# Implementation: Real-Time Collaborative Editor (Java & WebSockets)

This guide provides a runnable implementation of the communication layer for a Google Docs-like application using **Java Spring Boot** and **STOMP over WebSockets**.

## 1. Project Setup (Dependencies)

If you are using Maven, add the standard WebSocket starter to your `pom.xml`.

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-websocket</artifactId>
    </dependency>
    
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

---

## 2. WebSocket Configuration

We use **STOMP (Simple Text Oriented Messaging Protocol)**. This creates a Pub/Sub model where users subscribe to "Topics" (Documents) and publish messages to "Application" destinations.

### `WebSocketConfig.java`

```java
package com.example.docs.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.messaging.simp.config.MessageBrokerRegistry;
import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;

@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        // 1. Enable a simple in-memory message broker.
        // Messages sent to "/topic/..." will be routed to connected clients.
        config.enableSimpleBroker("/topic");

        // 2. Application prefix.
        // Messages sent to "/app/..." will be routed to @MessageMapping methods in controllers.
        config.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        // The HTTP URL endpoint clients connect to for the initial handshake.
        // e.g., ws://localhost:8080/ws
        registry.addEndpoint("/ws")
                .setAllowedOriginPatterns("*") // Allow all origins for dev/testing
                .withSockJS(); // Enable SockJS fallback options
    }
}
```

---

## 3. Data Model (The Operation)

We do not send the full text on every keypress. We send the **Delta** (Change).

### `EditRequest.java`

```java
package com.example.docs.model;

public class EditRequest {
    private String type;      // "INSERT", "DELETE", "CURSOR_MOVE"
    private int position;     // The index where the edit occurred
    private String payload;   // The character/string added
    private String userId;    // Who made the edit
    private int version;      // Document version for optimistic locking/OT

    // Default Constructor
    public EditRequest() {}

    public EditRequest(String type, int position, String payload, String userId, int version) {
        this.type = type;
        this.position = position;
        this.payload = payload;
        this.userId = userId;
        this.version = version;
    }

    // Getters and Setters
    public String getType() { return type; }
    public void setType(String type) { this.type = type; }

    public int getPosition() { return position; }
    public void setPosition(int position) { this.position = position; }

    public String getPayload() { return payload; }
    public void setPayload(String payload) { this.payload = payload; }

    public String getUserId() { return userId; }
    public void setUserId(String userId) { this.userId = userId; }
    
    public int getVersion() { return version; }
    public void setVersion(int version) { this.version = version; }
}
```

---

## 4. Document State Manager (Service Layer)

This acts as the "Collaboration Service" discussed in the system design. It holds the active state of documents in memory.

### `DocumentService.java`

```java
package com.example.docs.service;

import com.example.docs.model.EditRequest;
import org.springframework.stereotype.Service;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Service
public class DocumentService {

    // Simulating the "Hot" database/cache (Redis).
    // Key: DocID, Value: The current text content of the document.
    private final Map<String, StringBuilder> documentStore = new ConcurrentHashMap<>();

    /**
     * Applies the incoming edit to the server's in-memory state.
     */
    public void handleEdit(String docId, EditRequest edit) {
        // Initialize doc if not exists
        documentStore.putIfAbsent(docId, new StringBuilder(""));
        StringBuilder doc = documentStore.get(docId);

        // SYNCHRONIZATION:
        // Even though ConcurrentHashMap is thread-safe, the StringBuilder operations 
        // are not atomic. We lock the specific document object to prevent race conditions 
        // between two users editing the same doc simultaneously.
        synchronized (doc) {
            
            // --- OT Logic Placeholder ---
            // In a production system, you would check edit.getVersion() here.
            // If the version is old, you would transform 'edit.position'.
            
            if ("INSERT".equalsIgnoreCase(edit.getType())) {
                // Bounds check to avoid IndexOutOfBoundsException
                int pos = Math.min(edit.getPosition(), doc.length());
                doc.insert(pos, edit.getPayload());
            } 
            else if ("DELETE".equalsIgnoreCase(edit.getType())) {
                int start = Math.min(edit.getPosition(), doc.length());
                // Simple logic: delete 1 char. 
                if (start < doc.length()) {
                    doc.deleteCharAt(start);
                }
            }
        }
    }

    public String getContent(String docId) {
        return documentStore.getOrDefault(docId, new StringBuilder("")).toString();
    }
}
```

---

## 5. The Controller (API Endpoint)

This handles the message routing.

### `CollaborationController.java`

```java
package com.example.docs.controller;

import com.example.docs.model.EditRequest;
import com.example.docs.service.DocumentService;
import org.springframework.messaging.handler.annotation.DestinationVariable;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.stereotype.Controller;

@Controller
public class CollaborationController {

    private final DocumentService documentService;

    public CollaborationController(DocumentService documentService) {
        this.documentService = documentService;
    }

    /**
     * 1. Client sends message to: /app/edit/{docId}
     * 2. Server processes the edit.
     * 3. Server automatically broadcasts result to: /topic/doc/{docId}
     */
    @MessageMapping("/edit/{docId}")
    @SendTo("/topic/doc/{docId}")
    public EditRequest processEdit(@DestinationVariable String docId, EditRequest request) {
        System.out.println("Processing edit for Doc: " + docId + " from User: " + request.getUserId());

        // Update Server State
        documentService.handleEdit(docId, request);

        // Return the request to broadcast it to all subscribers.
        // NOTE: In a real OT system, we would return the *transformed* request, 
        // so clients can adjust their local state correctly.
        return request; 
    }
}
```

---

## 6. Client-Side Test (JavaScript)

You can use `stomp.js` and `sockjs-client` to test this in a browser console or simple HTML page.

```javascript
// 1. Connect
var socket = new SockJS('http://localhost:8080/ws');
var stompClient = Stomp.over(socket);

stompClient.connect({}, function (frame) {
    console.log('Connected: ' + frame);

    // 2. Subscribe to Document 123
    stompClient.subscribe('/topic/doc/123', function (messageOutput) {
        var edit = JSON.parse(messageOutput.body);
        console.log("INCOMING EDIT:", edit);
        
        // Logic to update your UI (e.g., CodeMirror or TextArea) goes here
        // applyEditToView(edit);
    });
});

// 3. Function to send an edit (Simulating typing 'A' at index 0)
function sendTestEdit() {
    stompClient.send("/app/edit/123", {}, JSON.stringify({
        'type': 'INSERT',
        'payload': 'A',
        'position': 0,
        'userId': 'User_A',
        'version': 1
    }));
}
```

---

## 7. Explanation of Mechanics

1.  **Protocol (STOMP):** We use STOMP instead of raw WebSockets because it provides a standard way to define "channels" (like `/topic/doc/123`). Without STOMP, we would have to manually parse JSON strings to figure out which document a user is editing.
2.  **Concurrency:** The `DocumentService` uses `synchronized(doc)` blocks. This is a simple form of concurrency control. If 100 users edit the same document, their requests are queued and applied one by one to the `StringBuilder` to ensure the internal server state doesn't get corrupted.
3.  **Broadcast:** The `@SendTo` annotation is key. It acts as a "Chat Room" for the document. Anything returned by `processEdit` is immediately pushed to every other user looking at that document.