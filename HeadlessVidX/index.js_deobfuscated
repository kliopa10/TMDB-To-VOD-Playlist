const express = require("express");
const { chromium, firefox, webkit } = require("playwright-extra");
const StealthPlugin = require("puppeteer-extra-plugin-stealth");
const { format } = require("date-fns");
const UserAgent = require("user-agents");
const https = require("https");
const http = require("http");
const fs = require("fs");
const path = require("path");
const { URL } = require("url");
const axios = require("axios");
const { exec } = require("child_process");
const WebSocket = require("ws");
const { Buffer } = require("buffer");

// Initialize Express app and WebSocket server
const app = express();
const stealthy = StealthPlugin();
const defaultPort = 3202;
const port = process.env.PORT || getPortFromFile() || defaultPort;
const server = http.createServer(app);
const wss = new WebSocket.Server({ server });

// Logging setup
const logFilePath = path.resolve(__dirname, "logs/combined.log");
const logDir = path.dirname(logFilePath);
if (!fs.existsSync(logDir)) {
  fs.mkdirSync(logDir, { recursive: true });
}
let logStream = fs.createWriteStream(logFilePath, { flags: "a" });
let clients = [];
const logs = new Map();
let currentRequestId = null;
let requestId, thisHost, thisPort;

// Override console.log to log to file and WebSocket clients
const originalConsoleLog = console.log;
console.log = (...args) => {
  const message = args.join(" ");
  const timestamp = format(new Date(), "MM/dd/yyyy hh:mm:ss a");
  const logMessage = `${timestamp} - ${message}\n`;
  originalConsoleLog.apply(console, args);
  if (currentRequestId) {
    const requestLogs = logs.get(currentRequestId) || [];
    requestLogs.push(logMessage.trim());
    logs.set(currentRequestId, requestLogs);
    if (requestId) broadcastLogs(currentRequestId);
  }
  logStream.write(logMessage);
};

// Broadcast logs to WebSocket clients
const broadcastLogs = (requestId) => {
  const requestLogs = logs.get(requestId);
  if (requestLogs) {
    const logLines = requestLogs.join("\n");
    clients.forEach((client) => {
      if (client.readyState === WebSocket.OPEN) {
        client.send(logLines);
      }
    });
    logStream.write(logLines + "\n");
    logs.delete(requestId);
  }
};

// WebSocket event handlers
wss.on("connection", (ws) => {
  console.log("Client connected");
  clients.push(ws);
  ws.on("close", () => {
    console.log("Client disconnected");
    clients = clients.filter((client) => client !== ws);
  });
});

wss.on("error", (error) => {
  console.log("WebSocket Server Error:", error);
});

// Log file size management
const MAX_LOG_SIZE = 10 * 1024 * 1024; // 10MB
function checkAndTruncateLog() {
  fs.stat(logFilePath, (err, stats) => {
    if (err) {
      console.error("Error checking log file size:", err);
      return;
    }
    if (stats.size > MAX_LOG_SIZE) {
      console.log("Truncating log file as it exceeds maximum size");
      fs.truncate(logFilePath, 0, (err) => {
        if (err) {
          console.error("Error truncating log file:", err);
        } else {
          console.log("Log file truncated successfully");
        }
      });
    }
  });
}
setInterval(checkAndTruncateLog, 60000); // Check every minute

// Download EasyList for ad-blocking
downloadEasyList();
const blockedDomains = loadBlockedDomains(path.join(__dirname, "blacklist/adblock-easylist.txt"));

// Helper functions
function getPortFromFile() {
  try {
    const portFilePath = path.join(__dirname, "listening-port.txt");
    const portFileContent = fs.readFileSync(portFilePath, "utf8").trim();
    const portNumber = parseInt(portFileContent, 10);
    if (!isNaN(portNumber) && portNumber > 0 && portNumber <= 65535) {
      return portNumber;
    } else {
      throw new Error(`Invalid port number in listening-port.txt: ${portFileContent}`);
    }
  } catch (err) {
    console.error(`Error reading port from listening-port.txt: ${err.message}.`);
    savePortToFile(defaultPort);
    return null;
  }
}

function savePortToFile(portNumber) {
  const portFilePath = path.join(__dirname, "listening-port.txt");
  try {
    fs.writeFileSync(portFilePath, portNumber.toString(), "utf8");
    console.log(`Port number ${portNumber} saved to listening-port.txt`);
  } catch (err) {
    console.error(`Error writing port to listening-port.txt: ${err.message}.`);
  }
}

// Middleware
app.use(express.json());

// Routes
app.get("/client", (req, res) => {
  res.sendFile(path.join(__dirname, "pages/logging.html"));
});

app.post("/extract-video", async (req, res) => {
  const {
    url,
    timeout,
    selectors,
    stealth,
    showBrowser,
    frame,
    ref,
    block,
    proxy,
    adblock,
    mustContain,
    notContain,
    blistjs,
    cusUseragent,
    browserType,
    bruteClick,
  } = req.body;

  requestId = Date.now().toString();
  currentRequestId = requestId;
  logs.set(requestId, []);

  const requestUrl = `${req.protocol}://${req.get("host")}${req.originalUrl}`;
  const urlObj = new URL(requestUrl);
  thisHost = urlObj.hostname;
  thisPort = urlObj.port || (req.secure ? 443 : 80);

  try {
    await startTask(
      url,
      timeout,
      selectors,
      stealth,
      showBrowser,
      frame,
      ref,
      block,
      proxy,
      adblock,
      res,
      mustContain,
      notContain,
      blistjs,
      cusUseragent,
      browserType,
      bruteClick,
      requestId
    );
    if (!res.headersSent) {
      res.status(200).json({ status: "success", message: "Task completed successfully" });
    }
  } catch (err) {
    if (!res.headersSent) {
      res.status(500).json({ status: "error", error: err.message });
    }
  }
});

// Start the server
server.listen(port, () => {
  console.log(`Server running at http://localhost:${port}/`);
  console.log(`WebSocket server is listening on port ${port}`);
});

// Helper functions for browser automation
async function startTask(
  url,
  timeout,
  selectors,
  stealth,
  showBrowser,
  frame,
  ref,
  block,
  proxy,
  adblock,
  res,
  mustContain,
  notContain,
  blistjs,
  cusUseragent,
  browserType,
  bruteClick,
  requestId
) {
  let browser;
  try {
    const debug = true;
    const timeoutValue = timeout ? parseInt(timeout) : 30000; // Default timeout: 30 seconds
    const selectorList = selectors ? selectors.split(",") : [];
    const isHeadless = showBrowser === "false";
    const isFrame = frame === "true";
    const isAdBlockEnabled = adblock === "true";
    const mustContainList = mustContain ? mustContain.split(",") : [];
    const notContainList = notContain ? notContain.split(",") : [];
    const userAgent = !cusUseragent || cusUseragent.trim().toLowerCase() === "random" || cusUseragent.trim().length === 0
      ? new UserAgent().toString()
      : cusUseragent.trim();
    const browserTypeValue = browserType || "chromium";

    // Launch browser
    browser = await launchBrowser(browserTypeValue, isHeadless, stealth, proxy);
    if (!browser) {
      throw new Error("Failed to launch browser");
    }

    // Set up browser context
    const context = await browser.newContext({
      userAgent,
      viewport: { width: 1280, height: 800 },
      locale: "en-US",
      timezoneId: "America/New_York",
    });
    const page = await context.newPage();

    // Block popups
    await blockPopups(context, page, debug);

    // Navigate to the URL
    await page.goto(url, { waitUntil: "domcontentloaded", timeout: timeoutValue });

    // Extract video URL
    const videoUrl = await extractVideoUrl(page, mustContainList, notContainList);
    if (videoUrl) {
      res.status(200).json({ status: "success", videoUrl });
    } else {
      res.status(404).json({ status: "error", message: "No video found" });
    }
  } catch (err) {
    console.error("Error during task execution:", err.message);
    if (!res.headersSent) {
      res.status(500).json({ status: "error", error: err.message });
    }
  } finally {
    if (browser) {
      await browser.close();
    }
  }
}

// Launch browser based on type
async function launchBrowser(browserType, isHeadless, stealth, proxy) {
  const launchOptions = {
    headless: isHeadless !== "false",
    args: ["--no-sandbox", "--disable-setuid-sandbox", "--disable-infobars", "--ignore-certificate-errors"],
  };

  if (proxy) {
    launchOptions.args.push(`--proxy-server=${proxy}`);
  }

  try {
    switch (browserType.toLowerCase()) {
      case "firefox":
        if (stealth) {
          launchOptions.firefoxUserPrefs = { "privacy.resistFingerprinting": true };
        }
        return await firefox.launch(launchOptions);
      case "chrome":
        if (stealth) chromium.use(stealthy);
        launchOptions.channel = "chrome";
        return await chromium.launch(launchOptions);
      case "chromium":
        if (stealth) chromium.use(stealthy);
        return await chromium.launch(launchOptions);
      case "webkit":
        if (stealth) webkit.use(stealthy);
        return await webkit.launch(launchOptions);
      default:
        throw new Error(`Unsupported browser type: ${browserType}`);
    }
  } catch (err) {
    console.error("Error launching browser:", err.message);
    return null;
  }
}

// Block popups
async function blockPopups(context, page, debug = false) {
  context.on("page", async (popup) => {
    if (popup !== page) {
      if (debug) console.log("Popup detected and closed!");
      try {
        await popup.close();
      } catch (err) {
        if (debug) console.error("Error closing popup:", err.message);
      }
    }
  });
}

// Extract video URL from page
async function extractVideoUrl(page, mustContainList, notContainList) {
  let videoUrl = null;
  page.on("response", async (response) => {
    const url = response.url();
    const contentType = response.headers()["content-type"];
    if (contentType && (contentType.includes("video") || contentType.includes("mpegurl"))) {
      if (
        (mustContainList.length === 0 || mustContainList.some((term) => url.includes(term))) &&
        (notContainList.length === 0 || !notContainList.some((term) => url.includes(term)))
      ) {
        videoUrl = url;
      }
    }
  });
  await page.waitForTimeout(5000); // Wait for responses
  return videoUrl;
}

// Download EasyList
function downloadEasyList() {
  const easylistUrl = "https://easylist-downloads.adblockplus.org/easylist.txt";
  const easylistPath = path.join(__dirname, "blacklist/adblock-easylist.txt");
  const easylistDir = path.dirname(easylistPath);
  if (!fs.existsSync(easylistDir)) {
    fs.mkdirSync(easylistDir, { recursive: true });
  }
  const easylistFile = fs.createWriteStream(easylistPath);
  https.get(easylistUrl, (response) => {
    if (response.statusCode !== 200) {
      console.error(`Failed to download EasyList. Status code: ${response.statusCode}`);
      return;
    }
    response.pipe(easylistFile);
    easylistFile.on("finish", () => {
      easylistFile.close(() => {
        console.log("Updated Adblock easylist.");
      });
    });
  }).on("error", (err) => {
    console.error("Error downloading EasyList:", err.message);
  });
}

// Load blocked domains from EasyList
function loadBlockedDomains(filePath) {
  const fileContent = fs.readFileSync(filePath, "utf8");
  const domainRegex = /^\|\|([^\^]+)\^/gm;
  const blockedDomains = new Set();
  let match;
  while ((match = domainRegex.exec(fileContent)) !== null) {
    blockedDomains.add(match[1]);
  }
  return blockedDomains;
}
