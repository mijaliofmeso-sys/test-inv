<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Inventory Adjuster with Project Tracking</title>
    <style>
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; text-align: center; padding: 20px; background: #f8f9fa; margin: 0; }
        .wrapper { background: #ffffff; padding: 35px 25px; border-radius: 16px; box-shadow: 0 4px 20px rgba(0,0,0,0.08); max-width: 380px; margin: 40px auto; border: 1px solid #eee; }
        h2 { color: #111; margin-bottom: 5px; }
        p { color: #666; font-size: 14px; margin-bottom: 25px; }
        
        /* Input Field Styling */
        .project-section { text-align: left; margin-bottom: 20px; }
        .project-section label { font-size: 13px; font-weight: bold; color: #444; text-transform: uppercase; display: block; margin-bottom: 6px; }
        .project-section input { width: 100%; padding: 12px; font-size: 15px; border: 1px solid #ccc; border-radius: 8px; box-sizing: border-box; background: #fafafa; }
        .project-section input:focus { border-color: #007bff; outline: none; background: #fff; }

        button { background: #007bff; color: white; border: none; padding: 16px 20px; font-size: 16px; font-weight: bold; border-radius: 8px; cursor: pointer; width: 100%; transition: background 0.2s; }
        button:disabled { background: #dcdcdc; color: #999; cursor: not-allowed; }
        
        /* Modal Styles */
        .modal { display: none; position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.5); align-items: center; justify-content: center; z-index: 100; }
        .modal-content { background: white; padding: 30px 20px; border-radius: 12px; width: 85%; max-width: 320px; text-align: center; box-shadow: 0 4px 15px rgba(0,0,0,0.2); }
        input[type="number"] { width: 80%; padding: 12px; font-size: 20px; margin: 15px 0; text-align: center; border: 2px solid #ccc; border-radius: 8px; }
        .modal-btns { display: flex; gap: 10px; margin-top: 10px; }
        .btn-confirm { background: #28a745; }
        .btn-cancel { background: #6c757d; }
        
        #status { margin-top: 25px; font-size: 15px; color: #444; font-weight: 500; line-height: 1.4; min-height: 45px; }

        /* History Log Styles */
        .history-box { margin-top: 30px; border-top: 2px dashed #eee; padding-top: 20px; text-align: left; }
        .history-title { font-size: 14px; font-weight: bold; color: #555; text-transform: uppercase; margin-bottom: 12px; letter-spacing: 0.5px; }
        .history-list { list-style: none; padding: 0; margin: 0; }
        .history-item { display: flex; justify-content: space-between; align-items: center; background: #fdfdfd; padding: 10px 14px; border-radius: 8px; margin-bottom: 8px; border: 1px solid #f0f0f0; font-size: 14px; }
        .history-details { font-family: monospace; }
        .badge { font-weight: bold; padding: 4px 8px; border-radius: 4px; font-size: 12px; font-family: monospace; }
        .badge-add { background: #e2f6ea; color: #28a745; }
        .badge-remove { background: #fde8e9; color: #dc3545; }
    </style>
</head>
<body>

<div class="wrapper">
    <div style="font-size: 50px; margin-bottom: 10px;">🔄</div>
    <h2>Inventory Adjuster</h2>
    <p>Assign a project code and tap a bin to update levels.</p>

    <!-- New Project Selection Textbox -->
    <div class="project-section">
        <label for="projectInput">Active Project Number / ID:</label>
        <input type="text" id="projectInput" placeholder="e.g., PROJ-2026-A" value="Internal Stock">
    </div>

    <button id="scanBtn">Activate NFC Antenna</button>
    <div id="status">System idle. Ready to configure details.</div>

    <div class="history-box">
        <div class="history-title">🕒 Session History (Last 3)</div>
        <ul id="historyList" class="history-list">
            <li style="color: #aaa; font-style: italic; list-style: none; text-align: center; font-size: 13px; padding: 10px 0;">No items scanned yet.</li>
        </ul>
    </div>
</div>

<!-- Quantity Input Modal Overlay -->
<div class="modal" id="qtyModal">
    <div class="modal-content">
        <h3 style="margin-top:0;">Bin Sensed!</h3>
        <div id="modalBinDisplay" style="font-weight:bold; color:#007bff; margin-bottom:10px;"></div>
        <label for="qtyInput">Enter adjustments:<br><small style="color:#777;">(Use positive numbers to add, negative numbers to deduct)</small></label>
        <br>
        <input type="number" id="qtyInput" placeholder="0" pattern="[0-9\-]*">
        <div class="modal-btns">
            <button class="btn-cancel" id="cancelBtn">Cancel</button>
            <button class="btn-confirm" id="submitBtn">Update Sheet</button>
        </div>
    </div>
</div>

<script>
    // PASTE YOUR RE-DEPLOYED GOOGLE APPS SCRIPT WEB APP URL HERE
    const GOOGLE_SCRIPT_URL = "https://script.google.com/macros/s/AKfycbwUIvRx_m0hW62YwqJ5HiGPPrnEO_xbtMkINUGggwWoDZuFU_IDXSsx_3VZP5T-NooXNw/exec";

    const scanBtn = document.getElementById('scanBtn');
    const statusDiv = document.getElementById('status');
    const qtyModal = document.getElementById('qtyModal');
    const qtyInput = document.getElementById('qtyInput');
    const projectInput = document.getElementById('projectInput');
    const modalBinDisplay = document.getElementById('modalBinDisplay');
    const submitBtn = document.getElementById('submitBtn');
    const cancelBtn = document.getElementById('cancelBtn');
    const historyList = document.getElementById('historyList');

    let currentActiveBinId = "";
    let localHistoryLog = [];

    scanBtn.addEventListener('click', async () => {
        if (!('NDEFReader' in window)) {
            statusDiv.innerHTML = "❌ <b>Web NFC Unsupported</b><br>Please use Google Chrome on an Android smartphone.";
            return;
        }

        try {
            statusDiv.innerHTML = "⏳ Booting device hardware...";
            const ndef = new NDEFReader();
            await ndef.scan();
            
            statusDiv.innerHTML = "📡 <b>Antenna Active!</b><br>Physically tap against a storage container chip...";
            scanBtn.disabled = true;

            ndef.addEventListener("readingerror", () => {
                statusDiv.innerHTML = "⚠️ Reading error. Realign your phone sensor with the chip.";
            });

            ndef.addEventListener("reading", ({ message, serialNumber }) => {
                let binId = serialNumber;
                
                for (const record of message.records) {
                    if (record.recordType === "text") {
                        const textDecoder = new TextDecoder(record.encoding);
                        binId = textDecoder.decode(record.data);
                    }
                }

                currentActiveBinId = binId;
                modalBinDisplay.innerText = `ID: ${binId}`;
                qtyInput.value = ""; 
                qtyModal.style.display = "flex"; 
                statusDiv.innerHTML = "Awaiting quantity input...";
            });

        } catch (error) {
            statusDiv.innerHTML = `❌ Hardware Error: ${error}`;
            scanBtn.disabled = false;
        }
    });

    submitBtn.addEventListener('click', () => {
        const value = parseInt(qtyInput.value, 10);
        if (isNaN(value) || value === 0) {
            alert("Please enter a valid non-zero transaction adjustment.");
            return;
        }
        
        qtyModal.style.display = "none";
        
        // Grab current value inside project textbox field
        const activeProject = projectInput.value.trim() || "Unassigned";
        transmitAdjustmentToCloud(currentActiveBinId, value, activeProject);
    });

    cancelBtn.addEventListener('click', () => {
        qtyModal.style.display = "none";
        statusDiv.innerHTML = "📡 <b>Antenna Active!</b><br>Ready to receive a new bin tap...";
    });

    async function transmitAdjustmentToCloud(binId, amount, projectCode) {
        statusDiv.innerHTML = `📤 Syncing details to cloud under project: [${projectCode}]...`;
        
        try {
            await fetch(GOOGLE_SCRIPT_URL, {
                method: "POST",
                mode: "no-cors", 
                headers: { "Content-Type": "application/json" },
                body: JSON.stringify({ binId: binId, adjustmentAmount: amount, projectNum: projectCode })
            });

            statusDiv.innerHTML = `✅ <b>Success!</b><br>Logged ${amount} units for Bin <b>${binId}</b> under <b>${projectCode}</b>.`;
            
            updateHistoryUI(binId, amount, projectCode);

        } catch (error) {
            statusDiv.innerHTML = `❌ Network Transmission Error: ${error}`;
        } finally {
            scanBtn.disabled = false;
        }
    }

    function updateHistoryUI(binId, amount, projectCode) {
        const newLogEntry = {
            bin: binId,
            change: amount,
            proj: projectCode,
            time: new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })
        };

        localHistoryLog.unshift(newLogEntry);

        if (localHistoryLog.length > 3) {
            localHistoryLog.pop();
        }

        historyList.innerHTML = "";
        
        localHistoryLog.forEach(item => {
            const li = document.createElement('li');
            li.className = "history-item";
            
            const badgeClass = item.change > 0 ? "badge badge-add" : "badge badge-remove";
            const prefix = item.change > 0 ? "+" : "";
            
            li.innerHTML = `
                <div class="history-details">
                    <strong>${item.bin}</strong> <span style="color:#666;">(${item.proj})</span><br>
                    <span style="font-size:11px; color:#999;">Logged at ${item.time}</span>
                </div>
                <span class="${badgeClass}">${prefix}${item.change}</span>
            `;
