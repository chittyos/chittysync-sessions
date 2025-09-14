# Platform-Specific Adapters for Cross-Session Sync

Unique adapters for every Claude and ChatGPT interface, leveraging their native capabilities.

## 🎭 Claude Platform Adapters

### 1. Claude Web (claude.ai)
```javascript
// claude-web-adapter.js
// Hijacks web console and DOM for coordination

(function() {
  // Inject into Claude's message handling
  const originalFetch = window.fetch;
  window.fetch = async function(...args) {
    const [url, options] = args;

    // Intercept Claude API calls
    if (url.includes('/api/append_message')) {
      const body = JSON.parse(options.body);

      // Check coordination file references
      if (body.prompt?.includes('COORDINATION') ||
          body.prompt?.includes('.ai-coordination')) {
        // Auto-inject session context
        body.prompt = `[SESSION-WEB-${Date.now()}]\n${body.prompt}`;

        // Store in localStorage for persistence
        localStorage.setItem('claude-session-id', `WEB-${Date.now()}`);

        // Sync to IndexedDB for larger storage
        const db = await openCoordinationDB();
        await db.put('sessions', {
          id: localStorage.getItem('claude-session-id'),
          platform: 'claude-web',
          timestamp: Date.now()
        });
      }

      options.body = JSON.stringify(body);
    }

    return originalFetch.apply(this, [url, options]);
  };

  // Monitor for coordination patterns in responses
  const observer = new MutationObserver((mutations) => {
    mutations.forEach((mutation) => {
      if (mutation.type === 'childList') {
        const text = mutation.target.textContent;
        if (text?.includes('[SESSION-') || text?.includes('.ai-coordination')) {
          // Trigger coordination sync
          syncCoordinationState();
        }
      }
    });
  });

  observer.observe(document.body, {
    childList: true,
    subtree: true
  });
})();
```

### 2. Claude Code (VS Code Extension)
```typescript
// claude-code-adapter.ts
// Native VS Code integration with file system access

import * as vscode from 'vscode';
import * as fs from 'fs';
import * as path from 'path';

export class ClaudeCodeCoordinator {
  private sessionId: string;
  private coordinationPath: string;
  private fileWatcher: vscode.FileSystemWatcher;

  constructor() {
    this.sessionId = `CODE-${Date.now().toString(36)}`;
    this.coordinationPath = path.join(
      vscode.workspace.rootPath || '',
      '.ai-coordination'
    );
  }

  activate(context: vscode.ExtensionContext) {
    // Register commands
    vscode.commands.registerCommand('claude.syncCoordination', () => {
      this.syncWithOtherSessions();
    });

    // Watch coordination files
    this.fileWatcher = vscode.workspace.createFileSystemWatcher(
      '**/.ai-coordination/**'
    );

    this.fileWatcher.onDidChange((uri) => {
      this.handleCoordinationUpdate(uri);
    });

    // Hook into Claude's file operations
    vscode.workspace.onDidSaveTextDocument((document) => {
      if (this.isCoordinationFile(document.uri)) {
        this.broadcastUpdate(document);
      }
    });

    // Auto-tag commits
    vscode.workspace.onWillSaveTextDocument((event) => {
      if (event.document.uri.path.endsWith('.git/COMMIT_EDITMSG')) {
        const edit = vscode.TextEdit.insert(
          new vscode.Position(0, 0),
          `[SESSION-${this.sessionId}] `
        );
        event.waitUntil(Promise.resolve([edit]));
      }
    });
  }

  private syncWithOtherSessions() {
    // Read all session files
    const sessions = fs.readdirSync(
      path.join(this.coordinationPath, 'sessions')
    );

    // Display in VS Code panel
    const panel = vscode.window.createWebviewPanel(
      'claudeCoordination',
      'Claude Coordination',
      vscode.ViewColumn.Two,
      {}
    );

    panel.webview.html = this.getCoordinationHTML(sessions);
  }
}
```

### 3. Claude Desktop App
```swift
// ClaudeDesktopAdapter.swift
// macOS native integration

import Foundation
import Combine

class ClaudeDesktopCoordinator {
    private let sessionId = "DESKTOP-\(UUID().uuidString.prefix(8))"
    private let fileManager = FileManager.default
    private var coordinationURL: URL
    private var fileMonitor: DispatchSourceFileSystemObject?

    init() {
        let homeURL = fileManager.homeDirectoryForCurrentUser
        coordinationURL = homeURL.appendingPathComponent(".ai-coordination")
        setupCoordination()
    }

    private func setupCoordination() {
        // Create coordination directory
        try? fileManager.createDirectory(
            at: coordinationURL,
            withIntermediateDirectories: true
        )

        // Register session
        let sessionFile = coordinationURL
            .appendingPathComponent("sessions")
            .appendingPathComponent("\(sessionId).json")

        let sessionData = [
            "id": sessionId,
            "platform": "claude-desktop",
            "pid": ProcessInfo.processInfo.processIdentifier,
            "timestamp": Date().timeIntervalSince1970
        ] as [String : Any]

        try? JSONSerialization.data(withJSONObject: sessionData)
            .write(to: sessionFile)

        // Setup file monitoring
        setupFileMonitor()

        // Setup distributed notifications
        DistributedNotificationCenter.default().addObserver(
            self,
            selector: #selector(handleCoordinationNotification),
            name: NSNotification.Name("ClaudeCoordinationUpdate"),
            object: nil
        )
    }

    private func setupFileMonitor() {
        let fd = open(coordinationURL.path, O_EVTONLY)
        fileMonitor = DispatchSource.makeFileSystemObjectSource(
            fileDescriptor: fd,
            eventMask: .write,
            queue: .main
        )

        fileMonitor?.setEventHandler { [weak self] in
            self?.handleCoordinationChange()
        }

        fileMonitor?.resume()
    }

    @objc private func handleCoordinationNotification(_ notification: Notification) {
        // Sync with other Claude instances
        if let data = notification.userInfo?["coordination"] as? Data {
            processCoordinationUpdate(data)
        }
    }
}
```

### 4. Claude iPhone App
```swift
// ClaudeIOSAdapter.swift
// iOS app with CloudKit sync

import UIKit
import CloudKit

class ClaudeIOSCoordinator {
    private let sessionId = "IOS-\(UUID().uuidString.prefix(8))"
    private let container = CKContainer(identifier: "iCloud.com.anthropic.claude")
    private let database: CKDatabase

    init() {
        database = container.privateCloudDatabase
        setupCoordination()
    }

    private func setupCoordination() {
        // Create session record in CloudKit
        let sessionRecord = CKRecord(recordType: "Session")
        sessionRecord["sessionId"] = sessionId
        sessionRecord["platform"] = "claude-ios"
        sessionRecord["device"] = UIDevice.current.name
        sessionRecord["timestamp"] = Date()

        database.save(sessionRecord) { record, error in
            if let error = error {
                print("Error saving session: \(error)")
            }
        }

        // Subscribe to coordination updates
        let subscription = CKQuerySubscription(
            recordType: "CoordinationUpdate",
            predicate: NSPredicate(value: true),
            options: [.firesOnRecordCreation, .firesOnRecordUpdate]
        )

        let notification = CKSubscription.NotificationInfo()
        notification.shouldSendContentAvailable = true
        subscription.notificationInfo = notification

        database.save(subscription) { _, _ in }
    }

    func handleClaudeMessage(_ message: String) -> String {
        // Check for coordination patterns
        if message.contains("COORDINATION") ||
           message.contains(".ai-coordination") {
            // Inject session context
            return "[SESSION-\(sessionId)] \(message)"
        }
        return message
    }
}
```

### 5. Claude Mac App
```swift
// ClaudeMacAdapter.swift
// macOS app with system integration

import AppKit
import Combine

class ClaudeMacCoordinator: NSObject {
    private let sessionId = "MACAPP-\(ProcessInfo.processInfo.globallyUniqueString.prefix(8))"
    private var scriptingBridge: SBApplication?

    override init() {
        super.init()
        setupSystemIntegration()
    }

    private func setupSystemIntegration() {
        // Register Apple Event handlers
        NSAppleEventManager.shared().setEventHandler(
            self,
            andSelector: #selector(handleCoordinationEvent),
            forEventClass: AEEventClass(kCoreEventClass),
            andEventID: AEEventID(kAEOpenDocuments)
        )

        // Setup Shortcuts integration
        if #available(macOS 12.0, *) {
            setupShortcutsIntegration()
        }

        // Monitor clipboard for coordination patterns
        Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { _ in
            self.checkClipboardForCoordination()
        }
    }

    @objc private func handleCoordinationEvent(_ event: NSAppleEventDescriptor) {
        // Handle coordination events from other apps
    }
}
```

## 🤖 ChatGPT Platform Adapters

### 1. ChatGPT Web
```javascript
// chatgpt-web-adapter.js
// Browser userscript for ChatGPT

// ==UserScript==
// @name         ChatGPT Coordination Adapter
// @namespace    http://tampermonkey.net/
// @version      1.0
// @description  Cross-session coordination for ChatGPT
// @match        https://chat.openai.com/*
// @grant        GM_xmlhttpRequest
// @grant        GM_setValue
// @grant        GM_getValue
// ==/UserScript==

(function() {
    'use strict';

    const SESSION_ID = `GPT-WEB-${Date.now().toString(36)}`;

    // Intercept message submission
    const originalSubmit = unsafeWindow.fetch;
    unsafeWindow.fetch = async function(...args) {
        const [url, options] = args;

        if (url.includes('/backend-api/conversation')) {
            const body = JSON.parse(options.body);

            // Check for coordination keywords
            if (body.messages?.[0]?.content?.parts?.[0]?.includes('COORDINATION')) {
                // Inject session identifier
                body.messages[0].content.parts[0] =
                    `[SESSION-${SESSION_ID}]\n${body.messages[0].content.parts[0]}`;

                // Store in GM storage
                GM_setValue('session_id', SESSION_ID);
                GM_setValue('last_coordination', Date.now());
            }

            options.body = JSON.stringify(body);
        }

        return originalSubmit.apply(this, args);
    };

    // Monitor responses for coordination
    const observer = new MutationObserver((mutations) => {
        mutations.forEach((mutation) => {
            const text = mutation.target.textContent;
            if (text?.includes('.ai-coordination')) {
                syncWithGitHub();
            }
        });
    });

    observer.observe(document.body, {
        childList: true,
        subtree: true
    });

    function syncWithGitHub() {
        GM_xmlhttpRequest({
            method: 'POST',
            url: 'https://api.github.com/repos/USER/REPO/dispatches',
            headers: {
                'Authorization': 'token ' + GM_getValue('github_token'),
                'Content-Type': 'application/json'
            },
            data: JSON.stringify({
                event_type: 'coordination_sync',
                client_payload: {
                    session_id: SESSION_ID,
                    platform: 'chatgpt-web'
                }
            })
        });
    }
})();
```

### 2. Custom GPT
```python
# custom_gpt_coordinator.py
# Custom GPT with Actions for coordination

import json
import hashlib
from datetime import datetime
from typing import Dict, Any

class CustomGPTCoordinator:
    def __init__(self):
        self.session_id = f"CUSTOM-GPT-{hashlib.md5(str(datetime.now()).encode()).hexdigest()[:8]}"
        self.coordination_state = {}

    def action_register_session(self) -> Dict[str, Any]:
        """OpenAI Action: Register this Custom GPT session"""
        return {
            "operationId": "registerSession",
            "description": "Register Custom GPT in coordination system",
            "requestBody": {
                "content": {
                    "application/json": {
                        "schema": {
                            "type": "object",
                            "properties": {
                                "session_id": {"type": "string"},
                                "gpt_name": {"type": "string"},
                                "capabilities": {"type": "array"}
                            }
                        }
                    }
                }
            }
        }

    def action_claim_task(self, task_id: str) -> Dict[str, Any]:
        """OpenAI Action: Claim a coordination task"""
        return {
            "operationId": "claimTask",
            "parameters": [{
                "name": "task_id",
                "in": "path",
                "required": True,
                "schema": {"type": "string"}
            }]
        }

    def action_sync_state(self) -> Dict[str, Any]:
        """OpenAI Action: Sync with other sessions"""
        return {
            "operationId": "syncState",
            "responses": {
                "200": {
                    "description": "Current coordination state",
                    "content": {
                        "application/json": {
                            "schema": {
                                "type": "object",
                                "properties": {
                                    "sessions": {"type": "array"},
                                    "tasks": {"type": "array"},
                                    "locks": {"type": "object"}
                                }
                            }
                        }
                    }
                }
            }
        }

# OpenAPI spec for Custom GPT
openapi_spec = {
    "openapi": "3.0.0",
    "info": {
        "title": "Cross-Session Coordination API",
        "version": "1.0.0"
    },
    "servers": [{
        "url": "https://cross-session-sync.workers.dev"
    }],
    "paths": {
        "/session/register": {
            "post": CustomGPTCoordinator().action_register_session()
        },
        "/task/{task_id}/claim": {
            "post": CustomGPTCoordinator().action_claim_task("")
        },
        "/sync": {
            "get": CustomGPTCoordinator().action_sync_state()
        }
    }
}
```

### 3. ChatGPT iPhone App
```swift
// ChatGPTIOSAdapter.swift
// iOS ChatGPT app coordination

import UIKit
import Network

class ChatGPTIOSCoordinator {
    private let sessionId = "GPT-IOS-\(UUID().uuidString.prefix(8))"
    private let monitor = NWPathMonitor()
    private var connection: NWConnection?

    init() {
        setupNetworkCoordination()
        setupAppGroupSharing()
    }

    private func setupNetworkCoordination() {
        // Connect to coordination WebSocket
        let endpoint = NWEndpoint.hostPort(
            host: "cross-session-sync.workers.dev",
            port: 443
        )

        let parameters = NWParameters.tls
        connection = NWConnection(to: endpoint, using: parameters)

        connection?.stateUpdateHandler = { state in
            if state == .ready {
                self.registerSession()
            }
        }

        connection?.start(queue: .main)
    }

    private func setupAppGroupSharing() {
        // Share coordination state via App Groups
        let sharedDefaults = UserDefaults(suiteName: "group.com.openai.chatgpt")
        sharedDefaults?.set(sessionId, forKey: "coordination_session_id")

        // Monitor shared container for updates
        let sharedContainer = FileManager.default
            .containerURL(forSecurityApplicationGroupIdentifier: "group.com.openai.chatgpt")

        if let coordinationURL = sharedContainer?.appendingPathComponent("coordination") {
            monitorDirectory(at: coordinationURL)
        }
    }

    private func registerSession() {
        let registration = [
            "session_id": sessionId,
            "platform": "chatgpt-ios",
            "device": UIDevice.current.model
        ]

        if let data = try? JSONSerialization.data(withJSONObject: registration) {
            connection?.send(content: data, completion: .contentProcessed { _ in })
        }
    }
}
```

### 4. ChatGPT Mac App
```swift
// ChatGPTMacAdapter.swift
// macOS ChatGPT app

import AppKit
import Combine

class ChatGPTMacCoordinator {
    private let sessionId = "GPT-MAC-\(UUID().uuidString.prefix(8))"
    private var cancellables = Set<AnyCancellable>()

    init() {
        setupCoordination()
        setupAutomatorIntegration()
    }

    private func setupCoordination() {
        // Use XPC for inter-process communication
        let connection = NSXPCConnection(serviceName: "com.openai.chatgpt.coordination")
        connection.remoteObjectInterface = NSXPCInterface(with: CoordinationProtocol.self)
        connection.resume()

        // Register with coordination service
        if let coordinator = connection.remoteObjectProxy as? CoordinationProtocol {
            coordinator.registerSession(sessionId, platform: "chatgpt-mac")
        }
    }

    private func setupAutomatorIntegration() {
        // Create Automator actions for coordination
        let script = """
            tell application "ChatGPT"
                set coordination_session to "\(sessionId)"
                -- Auto-inject session ID into prompts
            end tell
        """

        if let appleScript = NSAppleScript(source: script) {
            appleScript.executeAndReturnError(nil)
        }
    }
}
```

## 🌐 Browser Extensions

### Chrome Extension
```javascript
// manifest.json
{
  "manifest_version": 3,
  "name": "AI Coordination Extension",
  "version": "1.0",
  "permissions": ["storage", "tabs", "webRequest"],
  "host_permissions": [
    "https://claude.ai/*",
    "https://chat.openai.com/*"
  ],
  "background": {
    "service_worker": "background.js"
  },
  "content_scripts": [{
    "matches": ["https://claude.ai/*", "https://chat.openai.com/*"],
    "js": ["content.js"]
  }]
}

// content.js
const SESSION_ID = `EXT-CHROME-${Date.now().toString(36)}`;

// Detect which AI platform
const platform = window.location.hostname.includes('claude') ? 'claude' : 'chatgpt';

// Inject coordination layer
function injectCoordination() {
    const script = document.createElement('script');
    script.textContent = `
        window.AI_COORDINATION = {
            sessionId: '${SESSION_ID}',
            platform: '${platform}',
            syncState: ${syncState.toString()},
            claimTask: ${claimTask.toString()}
        };
    `;
    document.head.appendChild(script);
}

function syncState() {
    chrome.storage.sync.get(['coordination_state'], (result) => {
        window.postMessage({
            type: 'COORDINATION_SYNC',
            data: result.coordination_state
        }, '*');
    });
}

function claimTask(taskId) {
    chrome.runtime.sendMessage({
        action: 'claim_task',
        taskId: taskId,
        sessionId: SESSION_ID
    });
}

injectCoordination();
```

### Safari Extension
```swift
// SafariExtensionHandler.swift
import SafariServices

class SafariExtensionHandler: SFSafariExtensionHandler {
    let sessionId = "EXT-SAFARI-\(UUID().uuidString.prefix(8))"

    override func messageReceived(
        withName messageName: String,
        from page: SFSafariPage,
        userInfo: [String : Any]?
    ) {
        if messageName == "coordination_sync" {
            handleCoordinationSync(page: page, userInfo: userInfo)
        }
    }

    private func handleCoordinationSync(page: SFSafariPage, userInfo: [String : Any]?) {
        // Sync with iCloud
        let store = NSUbiquitousKeyValueStore.default
        store.set(sessionId, forKey: "coordination_session_id")
        store.set(Date(), forKey: "coordination_last_sync")
        store.synchronize()

        // Send update back to page
        page.dispatchMessageToScript(
            withName: "coordination_update",
            userInfo: ["sessionId": sessionId]
        )
    }
}
```

## 🔄 Universal Sync Protocol

All platforms communicate through:
1. **Cloudflare Workers** - Central coordination API
2. **GitHub Actions** - Event-driven workflows
3. **Neon Database** - Persistent state with branches
4. **Notion API** - Visual dashboards
5. **Platform-specific storage** - Local caching

Each adapter automatically:
- Registers its session on startup
- Maintains heartbeat
- Syncs coordination state
- Claims tasks atomically
- Handles platform-specific capabilities