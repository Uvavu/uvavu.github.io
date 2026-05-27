# uvavu.github.io
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Tooling Checkout System</title>
    <script src="https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4"></script>
    <script src="https://unpkg.com/html5-qrcode"></script>
</head>
<body class="bg-gray-100 font-sans p-4">

    <div class="max-w-md mx-auto bg-white rounded-xl shadow-md overflow-hidden p-6 space-y-6">
        <h1 class="text-2xl font-bold text-gray-900 text-center">🛠️ Tool Checkout</h1>

        <div class="space-y-3">
            <h2 class="text-lg font-semibold text-gray-700">1. User Info</h2>
            <div>
                <label class="block text-sm font-medium text-gray-600">User Name / ID</label>
                <input type="text" id="userName" class="mt-1 block w-full rounded-md border-gray-300 shadow-sm p-2 bg-gray-50 focus:ring-blue-500 focus:border-blue-500" placeholder="John Doe">
            </div>
            <div>
                <label class="block text-sm font-medium text-gray-600">Action</label>
                <select id="checkoutAction" class="mt-1 block w-full rounded-md border-gray-300 shadow-sm p-2 bg-gray-50">
                    <option value="Checkout">Checkout</option>
                    <option value="Check-in">Check-in</option>
                </select>
            </div>
        </div>

        <hr class="border-gray-200">

        <div class="space-y-3">
            <h2 class="text-lg font-semibold text-gray-700">2. Scan Tool QR Code</h2>
            <div id="reader" class="w-full bg-black rounded-lg overflow-hidden"></div>
            <p id="scanStatus" class="text-sm text-center text-gray-500 italic">Camera inactive</p>
        </div>

        <hr class="border-gray-200">

        <div class="space-y-3">
            <div class="flex justify-between items-center">
                <h2 class="text-lg font-semibold text-gray-700">3. Local Log</h2>
                <button onclick="exportToCSV()" class="bg-green-600 hover:bg-green-700 text-white text-xs font-bold py-1 px-3 rounded shadow">
                    Export CSV
                </button>
            </div>
            
            <div class="overflow-x-auto max-h-60 border border-gray-200 rounded-md">
                <table class="min-w-full divide-y divide-gray-200 text-sm">
                    <thead class="bg-gray-50 sticky top-0">
                        <tr>
                            <th class="px-3 py-2 text-left font-medium text-gray-500">Timestamp</th>
                            <th class="px-3 py-2 text-left font-medium text-gray-500">Tool ID</th>
                            <th class="px-3 py-2 text-left font-medium text-gray-500">User</th>
                            <th class="px-3 py-2 text-left font-medium text-gray-500">Action</th>
                        </tr>
                    </thead>
                    <tbody id="logTableBody" class="bg-white divide-y divide-gray-200">
                        </tbody>
                </table>
            </div>
            <button onclick="clearLog()" class="w-full bg-red-100 hover:bg-red-200 text-red-700 text-xs py-1 rounded">
                Clear Local Log
            </button>
        </div>
    </div>

    <script>
        let db;
        // 1. Initialize IndexedDB (Local Phone Storage)
        const request = indexedDB.open("ToolingSystemDB", 1);
        
        request.onupgradeneeded = function(e) {
            db = e.target.result;
            db.createObjectStore("logs", { keyPath: "id", autoIncrement: true });
        };

        request.onsuccess = function(e) {
            db = e.target.result;
            displayLogs(); // Load existing data on start
        };

        // 2. Initialize QR Code Scanner
        const html5QrcodeScanner = new Html5QrcodeScanner("reader", { 
            fps: 10, 
            qrbox: { width: 250, height: 250 } 
        });
        
        html5QrcodeScanner.render(onScanSuccess, onScanFailure);

        function onScanSuccess(decodedText, decodedResult) {
            const userName = document.getElementById("userName").value.trim();
            const action = document.getElementById("checkoutAction").value;
            
            if (!userName) {
                alert("Please enter a User Name before scanning.");
                return;
            }

            // Pause scanning briefly to prevent double-scans
            html5QrcodeScanner.pause(true);
            document.getElementById("scanStatus").innerText = "Processing scan...";

            const logEntry = {
                timestamp: new Date().toLocaleString(),
                toolId: decodedText,
                user: userName,
                action: action
            };

            // Save to IndexedDB
            const transaction = db.transaction(["logs"], "readwrite");
            const store = transaction.objectStore("logs");
            store.add(logEntry);

            transaction.oncomplete = function() {
                displayLogs();
                // Resume scanning after 2 seconds
                setTimeout(() => {
                    html5QrcodeScanner.resume();
                    document.getElementById("scanStatus").innerText = "Scanning active";
                }, 2000);
            };
        }

        function onScanFailure(error) {
            // Quietly handle scanning transitions, can be left empty
            document.getElementById("scanStatus").innerText = "Scanning active";
        }

        // 3. UI: Update Table from Local Storage
        function displayLogs() {
            const tbody = document.getElementById("logTableBody");
            tbody.innerHTML = "";
            
            const transaction = db.transaction(["logs"], "readonly");
            const store = transaction.objectStore("logs");
            
            // Open cursor to read data in reverse (newest first)
            store.openCursor(null, "prev").onsuccess = function(e) {
                const cursor = e.target.result;
                if (cursor) {
                    const row = `<tr>
                        <td class="px-3 py-2 text-gray-500 whitespace-nowrap">${cursor.value.timestamp}</td>
                        <td class="px-3 py-2 font-mono font-medium text-gray-900">${cursor.value.toolId}</td>
                        <td class="px-3 py-2 text-gray-700">${cursor.value.user}</td>
                        <td class="px-3 py-2"><span class="px-2 inline-flex text-xs leading-5 font-semibold rounded-full ${cursor.value.action === 'Checkout' ? 'bg-orange-100 text-orange-800' : 'bg-blue-100 text-blue-800'}">${cursor.value.action}</span></td>
                    </tr>`;
                    tbody.innerHTML += row;
                    cursor.continue();
                }
            };
        }

        // 4. Export Data to Spreadsheet format (CSV)
        function exportToCSV() {
            const transaction = db.transaction(["logs"], "readonly");
            const store = transaction.objectStore("logs");
            let csvContent = "data:text/csv;charset=utf-8,ID,Timestamp,Tool ID,User,Action\n";

            store.openCursor().onsuccess = function(e) {
                const cursor = e.target.result;
                if (cursor) {
                    const row = `"${cursor.value.id}","${cursor.value.timestamp}","${cursor.value.toolId}","${cursor.value.user}","${cursor.value.action}"\n`;
                    csvContent += row;
                    cursor.continue();
                } else {
                    // Trigger download
                    const encodedUri = encodeURI(csvContent);
                    const link = document.createElement("a");
                    link.setAttribute("href", encodedUri);
                    link.setAttribute("download", `tool_log_${new Date().toISOString().slice(0,10)}.csv`);
                    document.body.appendChild(link);
                    link.click();
                    document.body.removeChild(link);
                }
            };
        }

        // 5. Clear Database
        function clearLog() {
            if (confirm("Are you sure you want to clear all data on this device?")) {
                const transaction = db.transaction(["logs"], "readwrite");
                const store = transaction.objectStore("logs");
                store.clear().oncomplete = function() {
                    displayLogs();
                };
            }
        }
    </script>
</body>
</html>
