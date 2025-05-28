// ==UserScript==
// @name         jav-play
// @namespace    http://tampermonkey.net/
// @version      1.0
// @description  jav-play改油猴
// @author       jav-play
// @match        *://*.javdb.com/v/*
// @grant        GM.xmlHttpRequest
// @grant        GM.getValue
// @grant        GM.setValue
// @connect      missav.ws
// @run-at       document-idle
// ==/UserScript==

(function() {
    'use strict';

    const FEATURE_ENABLED_KEY = "javdb_helper_feature_enabled";
    const BUTTON_ID_IINA = "userscript-iina-floating-button";
    const BUTTON_ID_POTPLAYER = "userscript-potplayer-floating-button";

    // --- DOM Manipulation Functions ---
    function createOrUpdateButton(id, text, iconClass, href, topPosition) {
        let button = document.getElementById(id);
        if (!button) {
            button = document.createElement("a");
            button.id = id;
            button.className = "wxt-player-button"; // Re-using class for potential styling, or use a new one
            button.style.position = "fixed";
            button.style.right = "20px";
            button.style.padding = "10px";
            button.style.backgroundColor = "#3273dc"; // Example color, similar to JavDB buttons
            button.style.color = "white";
            button.style.textDecoration = "none";
            button.style.borderRadius = "4px";
            button.style.zIndex = "9999";
            button.style.fontSize = "14px";
            button.style.display = "flex";
            button.style.alignItems = "center";
            // Simple icon handling, can be improved with actual icon font or SVG
            button.innerHTML = `<span style="margin-right: 5px;">▶</span><span>${text}</span>`;
            document.body.appendChild(button);
            button.addEventListener("click", (event) => {
                event.preventDefault();
                window.location.href = button.href; // For protocol handlers like iina://
            });
        }
        button.href = href;
        button.style.top = topPosition;
        button.style.display = "flex";
    }

    function hideButton(id) {
        const button = document.getElementById(id);
        if (button) {
            button.style.display = "none";
        }
    }

    // --- Core Logic Functions ---

    function showPlayerButtons(uuid) {
        const m3u8Url = `https://surrit.com/${uuid}/playlist.m3u8`;
        const iinaLink = `iina://weblink?url=${encodeURIComponent(m3u8Url)}`;
        const potplayerLink = `potplayer://${m3u8Url}`; // PotPlayer might not need URI encoding for the full URL

        createOrUpdateButton(BUTTON_ID_IINA, "IINA", "icon-play", iinaLink, "100px");
        createOrUpdateButton(BUTTON_ID_POTPLAYER, "PotPlayer", "icon-play", potplayerLink, "150px");
    }

    function hideAllPlayerButtons() {
        hideButton(BUTTON_ID_IINA);
        hideButton(BUTTON_ID_POTPLAYER);
    }

    function extractBangouFromPage() {
        const copyButton = document.querySelector("a.button.is-white.copy-to-clipboard");
        if (!copyButton) {
            console.log("[JavDB Helper] Copy-to-clipboard button not found.");
            return "";
        }
        const bangou = copyButton.getAttribute("data-clipboard-text");
        if (bangou) {
            console.log("[JavDB Helper] Target Bangou:", bangou);
            return bangou;
        } else {
            console.log("[JavDB Helper] No Bangou found in data-clipboard-text.");
            return "";
        }
    }

    async function fetchMissavUuid(bangou) {
        const missavUrl = `https://missav.ws/dm1/en/${bangou.toLowerCase()}`;
        console.log("[JavDB Helper] Fetching from:", missavUrl);

        try {
            const response = await new Promise((resolve, reject) => {
                GM.xmlHttpRequest({
                    method: "GET",
                    url: missavUrl,
                    headers: {
                        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/90.0.4430.212 Safari/537.36"
                    },
                    onload: function(resp) {
                        if (resp.status >= 200 && resp.status < 400) { // Consider 3xx as potentially valid if redirects are handled
                            resolve({ success: true, html: resp.responseText });
                        } else {
                            resolve({ success: false, error: `Status: ${resp.status} on ${missavUrl}` });
                        }
                    },
                    onerror: function(resp) {
                        resolve({ success: false, error: `Error: ${resp.statusText || 'Network error'} on ${missavUrl}` });
                    },
                    ontimeout: function() {
                        resolve({ success: false, error: `Request timed out for ${missavUrl}` });
                    }
                });
            });

            if (!response.success) {
                throw new Error(response.error || "Failed to fetch from missav.ws");
            }

            const html = response.html;
            const parser = new DOMParser();
            const doc = parser.parseFromString(html, "text/html");
            const scripts = doc.getElementsByTagName("script");

            for (const scriptTag of scripts) {
                const scriptContent = scriptTag.textContent || "";
                if (scriptContent.includes("thumbnail") || scriptContent.includes("sources")) { // Broaden search slightly
                    const urlsMatch = scriptContent.match(/urls:\s*\[([^\]]*)\]/s) || scriptContent.match(/sources:\s*\[([^\]]*)\]/s);
                    if (urlsMatch && urlsMatch[1]) {
                        // Extract the first URL-like string, assuming it's the relevant one
                        const firstUrlPart = urlsMatch[1].split(",")[0].trim().replace(/["']/g, "").replace(/\\/g, "");
                        const uuidMatch = firstUrlPart.match(/\/([0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12})\//i) || // Standard UUID
                                          firstUrlPart.match(/\/([0-9a-fA-F-]+)\/seek\//i) || // Original regex
                                          firstUrlPart.match(/\/([0-9a-fA-F-]+)\/index\.m3u8/i); // Another common pattern

                        if (uuidMatch && uuidMatch[1]) {
                            console.log("[JavDB Helper] UUID Match:", uuidMatch[1]);
                            return uuidMatch[1];
                        }
                    }
                }
            }

            console.warn("[JavDB Helper] UUID not found in missav.ws response for", bangou);
            return "";

        } catch (error) {
            console.error("[JavDB Helper] Error fetching or parsing missav.ws:", error);
            return "";
        }
    }

    // --- Main Script Execution ---
    async function runScript() {
        const isFeatureEnabled = await GM.getValue(FEATURE_ENABLED_KEY, true);

        if (!isFeatureEnabled) {
            console.log("❌ [Jav Play] 功能已禁用。");
            hideAllPlayerButtons();
            // Optionally, add a button/menu item to re-enable it via GM.setValue
            return;
        }

        console.log("🚀 [Jav Play] 功能已启用，正在运行脚本...");

        const processPage = async () => {
            if (window.location.pathname.startsWith("/v/")) {
                const bangou = extractBangouFromPage();
                if (bangou) {
                    const uuid = await fetchMissavUuid(bangou);
                    if (uuid) {
                        showPlayerButtons(uuid);
                    } else {
                        hideAllPlayerButtons();
                    }
                } else {
                    hideAllPlayerButtons();
                }
            } else {
                hideAllPlayerButtons(); // Not on a video page
            }
        };

        // Initial execution
        processPage();

        // Observe DOM changes for single-page navigation on javdb
        let currentPathname = window.location.pathname;
        const observer = new MutationObserver(() => {
            if (window.location.pathname !== currentPathname) {
                currentPathname = window.location.pathname;
                console.log("[JavDB Helper] Path changed to:", currentPathname);
                processPage();
            }
        });

        observer.observe(document.body, { childList: true, subtree: true });
    }

    // --- Run the script ---
    runScript().catch(err => {
        console.error("[Jav Play] Critical error:", err);
    });

})();
