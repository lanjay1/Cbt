// ==UserScript==
// @name         Auto Jawab CBT AI - Google Gemini FREE
// @namespace    http://tampermonkey.net/
// @version      3.2
// @description  Auto jawab CBT pakai Google Gemini API (GRATIS!)
// @author       Gemini Version
// @match        *://*/*
// @grant        none
// @run-at       document-end
// ==/UserScript==

(function() {
    'use strict';

    // ═══════════════════════════════════════════════════════════
    // 🔑 EDIT API KEY DI SINI (WAJIB!)
    // ═══════════════════════════════════════════════════════════
    // Ambil API key gratis di: https://aistudio.google.com/apikey

    const GEMINI_API_KEY = 'AIzaSyAAWF5eXC4XVX1jXBnJ797EtWfRekQIN6Y';  

    // ═══════════════════════════════════════════════════════════
    // ⚙️ PENGATURAN (Opsional, bisa diubah sesuai kebutuhan)
    // ═══════════════════════════════════════════════════════════

    const CONFIG = {
        DELAY_MIN: 3000,        
        DELAY_MAX: 5000,        
        CLICK_DELAY: 1500,      
        AUTO_CLICK_NEXT: true   
    };

    // ═══════════════════════════════════════════════════════════
    // 🎯 SELECTOR HTML (Edit jika website berubah struktur)
    // ═══════════════════════════════════════════════════════════
    // Ini adalah selector CSS untuk mencari elemen di halaman
    // Edit bagian ini jika script tidak menemukan soal/pilihan

    const SELECTORS = {
        // Selector untuk elemen pertanyaan/soal
        question: [
            '.question-text',
            '.soal',
            '[class*="question"]',
            '[class*="soal"]',
            '.pertanyaan',
            '#question',
            '#soal',
            'h2',
            'h3'
        ].join(', '),

        // Selector untuk pilihan jawaban (radio button)
        options: [
            'input[type="radio"]',
            '.option input',
            '[class*="option"] input',
            '[name*="jawaban"]',
            '[name*="answer"]',
            '[class*="pilihan"] input'
        ].join(', '),

        // Selector untuk tombol next/submit
        nextButton: [
            '.next-btn',
            '.btn-next',
            'button[type="submit"]',
            '[class*="next"]',
            '[class*="submit"]',
            '[class*="lanjut"]',
            'button:contains("Selanjutnya")',
            'button:contains("Next")',
            'button:contains("Submit")'
        ].join(', ')
    };

    // ═══════════════════════════════════════════════════════════
    // 🚀 KODE UTAMA (Jangan edit bagian ini kecuali paham JS)
    // ═══════════════════════════════════════════════════════════

    let autoRunning = false;
    let stats = { total: 0, success: 0, failed: 0 };

    // Fungsi memanggil Google Gemini API
    async function callGeminiAPI(prompt) {
        if (GEMINI_API_KEY === 'YOUR_GEMINI_API_KEY_HERE') {
            throw new Error('❌ API Key belum diisi! Edit script dan masukkan Gemini API Key.');
        }

        const url = `https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=${GEMINI_API_KEY}`;

        try {
            const response = await fetch(url, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({
                    contents: [{
                        parts: [{
                            text: `Kamu adalah asisten ujian yang cerdas. Tugas kamu adalah menjawab soal pilihan ganda dengan HANYA SATU HURUF (A, B, C, D, atau E). Jangan berikan penjelasan, hanya huruf jawaban.\n\n${prompt}`
                        }]
                    }],
                    generationConfig: {
                        temperature: 0.1,
                        maxOutputTokens: 10,
                        topP: 1,
                        topK: 1
                    }
                })
            });

            if (!response.ok) {
                const error = await response.json();
                throw new Error(`Gemini API Error: ${error.error?.message || response.statusText}`);
            }

            const data = await response.json();
            return data.candidates[0].content.parts[0].text.trim();
        } catch (error) {
            console.error('❌ Error memanggil Gemini:', error);
            throw error;
        }
    }

    // Extract huruf jawaban dari respons AI
    function extractAnswer(text) {
        const match = text.toUpperCase().match(/[A-E]/);
        return match ? match[0] : null;
    }

    // Fungsi utama auto answer
    async function runAutoAnswer() {
        if (!autoRunning) return;

        // Cari elemen soal
        const questionEl = document.querySelector(SELECTORS.question);
        const options = Array.from(document.querySelectorAll(SELECTORS.options));
        const nextBtn = document.querySelector(SELECTORS.nextButton);

        // Cek apakah sudah dijawab
        const selected = options.find(opt => opt.checked);
        if (selected && nextBtn && CONFIG.AUTO_CLICK_NEXT) {
            console.log('✓ Sudah dijawab, klik next...');
            nextBtn.click();
            setTimeout(runAutoAnswer, CONFIG.DELAY_MIN + Math.random() * (CONFIG.DELAY_MAX - CONFIG.DELAY_MIN));
            return;
        }

        // Tunggu jika soal belum muncul
        if (!questionEl || options.length < 2) {
            console.log('⏸ Menunggu soal muncul...');
            setTimeout(runAutoAnswer, 2000);
            return;
        }

        // Ambil teks soal
        const question = questionEl.innerText.trim();

        // Ambil teks pilihan jawaban
        const optionTexts = options.map(opt => {
            const parent = opt.closest('label') || opt.parentElement;
            const label = parent?.innerText || opt.nextElementSibling?.innerText || opt.value || '';
            return label.trim();
        }).filter(t => t);

        if (optionTexts.length < 2) {
            console.log('⚠ Opsi jawaban tidak ditemukan');
            setTimeout(runAutoAnswer, 2000);
            return;
        }

        // Format prompt untuk AI
        const prompt = `Soal:\n${question}\n\nPilihan Jawaban:\n${optionTexts.map((t, i) =>
            `${String.fromCharCode(65 + i)}. ${t}`
        ).join('\n')}\n\nJawaban yang benar adalah:`;

        console.log('━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━');
        console.log('🤖 Mengirim ke Google Gemini...');
        console.log('📝 Soal:', question.substring(0, 150) + '...');
        updateStats('total');

        try {
            // Panggil AI
            const aiResponse = await callGeminiAPI(prompt);
            const answer = extractAnswer(aiResponse);

            if (!answer) {
                console.error('❌ AI tidak memberikan jawaban valid:', aiResponse);
                updateStats('failed');
                setTimeout(runAutoAnswer, CONFIG.DELAY_MIN);
                return;
            }

            const choiceIndex = answer.charCodeAt(0) - 65;

            if (choiceIndex >= 0 && choiceIndex < options.length) {
                console.log(`✅ AI Menjawab: ${answer}`);
                console.log(`📌 Jawaban: ${optionTexts[choiceIndex]}`);

                // Klik jawaban
                options[choiceIndex].click();
                updateStats('success');

                // Klik next setelah delay
                setTimeout(() => {
                    if (nextBtn && CONFIG.AUTO_CLICK_NEXT) {
                        nextBtn.click();
                        console.log('➡ Klik next button');
                    }
                    console.log('━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━\n');
                    setTimeout(runAutoAnswer, CONFIG.DELAY_MIN + Math.random() * (CONFIG.DELAY_MAX - CONFIG.DELAY_MIN));
                }, CONFIG.CLICK_DELAY + Math.random() * 1000);
            } else {
                console.error('❌ Jawaban diluar range:', answer);
                updateStats('failed');
                setTimeout(runAutoAnswer, CONFIG.DELAY_MIN);
            }
        } catch (error) {
            console.error('💥 Error:', error.message);

            // Jika API key salah, hentikan script
            if (error.message.includes('API Key')) {
                alert('⚠️ Google Gemini API Key salah atau belum diisi!\n\nCara mengisi:\n1. Buka https://aistudio.google.com/apikey\n2. Copy API Key\n3. Edit script di Tampermonkey\n4. Paste key di bagian GEMINI_API_KEY');
                autoRunning = false;
                toggleAuto();
                return;
            }

            updateStats('failed');
            setTimeout(runAutoAnswer, CONFIG.DELAY_MIN * 2);
        }
    }

    // ═══════════════════════════════════════════════════════════
    // 🎨 USER INTERFACE
    // ═══════════════════════════════════════════════════════════

    function updateStats(type) {
        stats[type]++;
        const btn = document.getElementById('cbt-auto-btn');
        if (btn && autoRunning) {
            const accuracy = stats.total > 0 ? Math.round((stats.success / stats.total) * 100) : 0;
            btn.textContent = `🤖 ON (${stats.success}/${stats.total} • ${accuracy}%)`;
        }
    }

    function createButton() {
        // Tombol utama
        const btn = document.createElement('button');
        btn.id = 'cbt-auto-btn';
        btn.textContent = '🤖 Auto AI (OFF)';
        btn.style.cssText = `
            position: fixed;
            top: 20px;
            right: 20px;
            z-index: 99999;
            padding: 12px 20px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            border: none;
            border-radius: 8px;
            cursor: pointer;
            font-weight: bold;
            font-size: 14px;
            box-shadow: 0 4px 15px rgba(0,0,0,0.2);
            transition: all 0.3s;
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
        `;

        btn.onmouseover = () => btn.style.transform = 'scale(1.05)';
        btn.onmouseout = () => btn.style.transform = 'scale(1)';
        btn.onclick = toggleAuto;
        document.body.appendChild(btn);

        // Info panel
        const info = document.createElement('div');
        info.id = 'cbt-auto-info';
        info.style.cssText = `
            position: fixed;
            top: 70px;
            right: 20px;
            z-index: 99998;
            padding: 12px;
            background: rgba(0,0,0,0.9);
            color: white;
            border-radius: 8px;
            font-size: 11px;
            max-width: 200px;
            box-shadow: 0 4px 15px rgba(0,0,0,0.3);
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
        `;
        info.innerHTML = `
            <div style="font-weight: bold; margin-bottom: 6px;">
                🤖 Google Gemini 1.5 Flash
            </div>
            <div style="font-size: 10px; opacity: 0.8; line-height: 1.4;">
                ✅ GRATIS<br>
                📊 Quota: 1500/hari<br>
                ⚡ Speed: ~2s/soal
            </div>
            <div style="margin-top: 8px; padding-top: 8px; border-top: 1px solid rgba(255,255,255,0.2); font-size: 10px; opacity: 0.7;">
                Klik tombol untuk mulai
            </div>
        `;
        document.body.appendChild(info);
    }

    function toggleAuto() {
        autoRunning = !autoRunning;
        const btn = document.getElementById('cbt-auto-btn');

        if (autoRunning) {
            // Cek API key terlebih dahulu
            if (GEMINI_API_KEY === 'YOUR_GEMINI_API_KEY_HERE') {
                alert('⚠️ API Key belum diisi!\n\nLangkah-langkah:\n1. Buka https://aistudio.google.com/apikey\n2. Login dengan Google\n3. Klik "Create API Key"\n4. Copy key yang didapat\n5. Edit script di Tampermonkey\n6. Paste di bagian GEMINI_API_KEY');
                return;
            }

            btn.textContent = '🤖 ON (0/0 • 0%)';
            btn.style.background = 'linear-gradient(135deg, #f093fb 0%, #f5576c 100%)';
            stats = { total: 0, success: 0, failed: 0 };
            console.log('🚀 ═══════════════════════════════════════');
            console.log('🚀 AUTO ANSWER AKTIF!');
            console.log('🚀 ═══════════════════════════════════════');
            runAutoAnswer();
        } else {
            btn.textContent = '🤖 Auto AI (OFF)';
            btn.style.background = 'linear-gradient(135deg, #667eea 0%, #764ba2 100%)';
            console.log('⏹ ═══════════════════════════════════════');
            console.log('⏹ AUTO ANSWER BERHENTI');
            console.log('📊 Total Soal:', stats.total);
            console.log('✅ Berhasil:', stats.success);
            console.log('❌ Gagal:', stats.failed);
            if (stats.total > 0) {
                console.log('🎯 Akurasi:', Math.round((stats.success / stats.total) * 100) + '%');
            }
            console.log('⏹ ═══════════════════════════════════════');
        }
    }

    // ═══════════════════════════════════════════════════════════
    // 🎬 INITIALIZE
    // ═══════════════════════════════════════════════════════════

    setTimeout(() => {
        createButton();
        console.log('✨ CBT Auto Answer loaded!');
        console.log('📍 URL matched:', window.location.href);
        console.log('🔑 API Key status:', GEMINI_API_KEY === 'YOUR_GEMINI_API_KEY_HERE' ? '❌ Belum diisi' : '✅ Sudah diisi');
    }, 1000);

});
