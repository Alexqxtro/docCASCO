<!DOCTYPE html>
<html lang="ro">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>IDP Daune Auto</title>
    <!-- Stilurile principale încărcate direct -->
    <script src="https://tailwindcss.com"></script>
    <script src="https://jsdelivr.net"></script>
</head>
<body class="bg-gray-100 text-gray-800 min-h-screen p-6">

    <div class="max-w-6xl mx-auto bg-white p-8 rounded-2xl shadow-md border border-gray-200">
        <!-- Header -->
        <div class="border-b pb-4 mb-6">
            <h1 class="text-2xl font-bold text-blue-900">Intelligent Document Processing (IDP)</h1>
            <p class="text-sm text-gray-600">Sistem real conectat la GPT-4o pentru validare dosare CASCO</p>
        </div>

        <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
            <!-- Zona din stanga -->
            <div class="md:col-span-2 space-y-4">
                <div class="border-2 border-dashed border-gray-300 rounded-xl p-6 text-center bg-gray-50">
                    <input type="file" id="fileInput" multiple accept="image/*,.pdf" class="mb-2 block mx-auto text-sm">
                    <p class="text-xs text-gray-500">Selectează poze sau documente PDF</p>
                </div>

                <button id="btnAnalyze" class="w-full bg-blue-600 hover:bg-blue-700 text-white font-semibold py-3 px-4 rounded-xl transition-all">
                    Analizează Documentele cu GPT-4o Real
                </button>

                <!-- Status de incarcare -->
                <div id="loadingOverlay" class="hidden text-center p-4 bg-blue-50 text-blue-700 rounded-xl font-medium animate-pulse">
                    AI analizează documentele tale prin serverul local... vă rugăm așteptați.
                </div>

                <!-- Tabel Rezultate -->
                <div id="resultsContainer" class="hidden border rounded-xl overflow-hidden bg-white">
                    <table class="w-full text-sm text-left">
                        <tr class="bg-gray-50 border-b text-xs uppercase font-semibold text-gray-600">
                            <th class="p-3">Fișier</th>
                            <th class="p-3">Tip identificat</th>
                            <th class="p-3">Status</th>
                            <th class="p-3">Observații AI</th>
                        </tr>
                        <tbody id="resultsTableBody"></tbody>
                    </table>
                </div>
            </div>

            <!-- Zona din dreapta: Checklist -->
            <div class="bg-gray-50 p-4 rounded-xl border">
                <h3 class="font-bold text-gray-900 mb-3 text-sm border-b pb-2">Checklist Documente</h3>
                <div id="checklistItems" class="space-y-2 text-xs font-medium text-gray-600"></div>
            </div>
        </div>
    </div>

    <script>
        const REQUIRED_DOCS = {
            "buletin": "Buletin / Carte de identitate",
            "permis_conducere": "Permis de conducere",
            "talon_auto": "Talon auto",
            "polita_casco": "Poliță CASCO",
            "contract_cesiune": "Contract cesiune",
            "contract_mandat": "Contract mandat",
            "imputernicire_proprietar": "Împuternicire proprietar",
            "fotografie_parbriz_avariat": "Foto parbriz avariat",
            "foto_cod_parbriz_avariat": "Foto cod parbriz avariat",
            "foto_serie_vin": "Foto serie VIN",
            "foto_parbriz_inlocuit_cu_cod_nou": "Foto parbriz înlocuit nou"
        };

        const fileInput = document.getElementById('fileInput');
        const btnAnalyze = document.getElementById('btnAnalyze');
        const loadingOverlay = document.getElementById('loadingOverlay');
        const resultsContainer = document.getElementById('resultsContainer');
        const resultsTableBody = document.getElementById('resultsTableBody');
        const checklistItems = document.getElementById('checklistItems');

        function initChecklist() {
            checklistItems.innerHTML = '';
            Object.entries(REQUIRED_DOCS).forEach(([key, val]) => {
                checklistItems.innerHTML += `
                    <div id="check-${key}" class="p-2 border rounded bg-white flex justify-between items-center">
                        <span>${val}</span><span class="text-gray-400 font-bold">○</span>
                    </div>`;
            });
        }

        btnAnalyze.addEventListener('click', async () => {
            if(fileInput.files.length === 0) { alert('Te rog selectează cel puțin un fișier!'); return; }
            
            loadingOverlay.classList.remove('hidden');
            resultsContainer.classList.add('hidden');

            const formData = new FormData();
            Array.from(fileInput.files).forEach(file => { formData.append('files', file); });

            try {
                const response = await fetch('http://127.0.0', { method: 'POST', body: formData });
                const data = await response.json();
                
                if (data.success) {
                    resultsTableBody.innerHTML = '';
                    initChecklist();

                    data.analysis.documents.forEach(doc => {
                        resultsTableBody.innerHTML += `
                            <tr class="border-b hover:bg-gray-50">
                                <td class="p-3 font-medium">${doc.file_name}</td>
                                <td class="p-3">${REQUIRED_DOCS[doc.document_type] || doc.document_type}</td>
                                <td class="p-3 font-bold ${doc.validity_status === 'valid' ? 'text-green-600' : 'text-red-600'}">${doc.validity_status}</td>
                                <td class="p-3 text-xs text-gray-600">${doc.observations}</td>
                            </tr>`;
                        
                        const el = document.getElementById(`check-${doc.document_type}`);
                        if(el) {
                            el.className = `p-2 border rounded flex justify-between items-center ${doc.validity_status === 'valid' ? 'bg-green-50 border-green-300 text-green-800' : 'bg-red-50 border-red-300 text-red-800'}`;
                            el.querySelector('span:last-child').innerHTML = doc.validity_status === 'valid' ? '✓' : '⚠️';
                        }
                    });

                    if(data.analysis.missing_documents) {
                        data.analysis.missing_documents.forEach(key => {
                            const el = document.getElementById(`check-${key}`);
                            if(el) { el.className = "p-2 border rounded flex justify-between items-center bg-red-50 border-red-200 text-red-700"; el.querySelector('span:last-child').innerHTML = '✗'; }
                        });
                    }
                    resultsContainer.classList.remove('hidden');
                } else {
                    alert('Eroare AI: ' + data.error);
                }
            } catch (err) {
                alert('Eroare: Nu s-a putut conecta la serverul local Python (http://127.0.0.1:5000). Porneste app.py in CMD.');
            } finally {
                loadingOverlay.classList.add('hidden');
            }
        });

        initChecklist();
    </script>
</body>
</html>
# docCASCO
