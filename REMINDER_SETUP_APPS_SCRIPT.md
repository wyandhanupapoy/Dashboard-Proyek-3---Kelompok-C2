# Pengingat Email Otomatis (Gratis) dengan Google Apps Script

Dokumen ini menjelaskan cara menyiapkan pengiriman email pengingat H- (hari sebelum deadline) secara otomatis tanpa biaya menggunakan Google Apps Script dan Gmail Anda.

## Ringkasan Arsitektur
- Aplikasi web (frontend ini) menyimpan tugas + pengingat di Firestore
- Opsional: Frontend juga mengirimkan salinan data tugas ke Web App Google Apps Script (endpoint HTTP) — cukup aktifkan dengan mengisi `APPS_SCRIPT_URL` di `index.html`
- Apps Script menyimpan data di Google Sheet dan menjalankan trigger terjadwal (time-driven) setiap hari untuk mengirim email pengingat yang jatuh tempo

Keuntungan: 100% gratis (menggunakan akun Google) dan berjalan otomatis di background tanpa harus membuka website.

## Langkah 1 — Siapkan Google Sheet
1. Buka Google Drive → New → Google Sheets → beri nama misal `C2 Tasks`
2. Pada baris pertama (header), tulis kolom berikut:
   - `id` | `taskName` | `picName` | `startDate` | `deadlineDate` | `priority` | `remindersJSON` | `sentJSON`

## Langkah 2 — Buat Google Apps Script Web App
1. Di Sheet: Extensions → Apps Script
2. Hapus isi editor, lalu tempel kode di bawah ini
3. File → Save (beri nama proyek)
4. Deploy → New deployment → Type: Web app
   - Description: `Webhook Upsert Tasks`
   - Execute as: Me
   - Who has access: Anyone with the link (atau Anyone)
   - Klik Deploy, salin URL Web App → tempel ke konstanta `APPS_SCRIPT_URL` di `index.html`

```javascript
// Spreadsheet target
const SHEET_NAME = 'Sheet1'; // ganti jika sheet tab Anda berbeda

function getSheet_() {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  return ss.getSheetByName(SHEET_NAME);
}

// Webhook untuk upsert task dari frontend
function doPost(e) {
  try {
    const payload = JSON.parse(e.postData.contents);
    if (payload.type === 'upsert-task') {
      upsertTask_(payload.task);
      return ContentService.createTextOutput(JSON.stringify({ ok: true })).setMimeType(ContentService.MimeType.JSON);
    }
    return ContentService.createTextOutput(JSON.stringify({ ok: false, error: 'Unknown type' })).setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService.createTextOutput(JSON.stringify({ ok: false, error: String(err) })).setMimeType(ContentService.MimeType.JSON);
  }
}

function upsertTask_(task) {
  const sh = getSheet_();
  const rows = sh.getDataRange().getValues();
  const header = rows[0];
  const idx = {
    id: header.indexOf('id'),
    taskName: header.indexOf('taskName'),
    picName: header.indexOf('picName'),
    startDate: header.indexOf('startDate'),
    deadlineDate: header.indexOf('deadlineDate'),
    priority: header.indexOf('priority'),
    remindersJSON: header.indexOf('remindersJSON'),
    sentJSON: header.indexOf('sentJSON'),
  };
  const findRow = () => {
    for (let r = 1; r < rows.length; r++) {
      if (String(rows[r][idx.id]) === String(task.id || '')) return r + 1; // 1-based row
    }
    return -1;
  };
  const row = findRow();
  const record = [
    task.id || '',
    task.taskName || '',
    task.picName || '',
    task.startDate || '',
    task.deadlineDate || '',
    task.priority || '',
    JSON.stringify(task.reminders || []),
    // sentJSON menyimpan penanda pengingat yang sudah dikirim, biarkan kosong jika baru
    row > 0 ? sh.getRange(row, idx.sentJSON + 1).getValue() : '{}'
  ];
  if (row > 0) {
    sh.getRange(row, 1, 1, record.length).setValues([record]);
  } else {
    sh.appendRow(record);
  }
}

// Utility untuk memeriksa apakah pengingat sudah pernah dikirim
function hasSent_(sentMap, key) {
  return Boolean(sentMap[key]);
}
function markSent_(sentMap, key) {
  sentMap[key] = new Date().toISOString();
}

// Kirim email harian berdasarkan H- pengingat
function sendReminders() {
  const sh = getSheet_();
  const rows = sh.getDataRange().getValues();
  if (rows.length <= 1) return;
  const header = rows[0];
  const idx = {
    id: header.indexOf('id'),
    taskName: header.indexOf('taskName'),
    picName: header.indexOf('picName'),
    startDate: header.indexOf('startDate'),
    deadlineDate: header.indexOf('deadlineDate'),
    priority: header.indexOf('priority'),
    remindersJSON: header.indexOf('remindersJSON'),
    sentJSON: header.indexOf('sentJSON'),
  };
  const today = new Date();
  today.setHours(0,0,0,0);

  for (let r = 1; r < rows.length; r++) {
    const id = rows[r][idx.id];
    const taskName = rows[r][idx.taskName];
    const picName = rows[r][idx.picName];
    const deadlineStr = rows[r][idx.deadlineDate];
    const reminders = JSON.parse(rows[r][idx.remindersJSON] || '[]');
    let sentMap = {};
    try { sentMap = JSON.parse(rows[r][idx.sentJSON] || '{}'); } catch (e) {}

    if (!deadlineStr) continue;
    const deadline = new Date(deadlineStr);
    deadline.setHours(0,0,0,0);

    reminders.forEach(rem => {
      const daysBefore = parseInt(rem.daysBefore || 0);
      const email = rem.email || '';
      if (!email) return;
      const due = new Date(deadline);
      due.setDate(deadline.getDate() - daysBefore);
      const key = `${email}|${daysBefore}|${deadline.toISOString().slice(0,10)}`;
      if (due.getTime() === today.getTime() && !hasSent_(sentMap, key)) {
        const subject = `[Pengingat Tugas] H-${daysBefore} • ${taskName}`;
        const body = `Halo,\n\nIni pengingat bahwa tugas:\n\n` +
          `• Nama: ${taskName}\n` +
          `• PIC: ${picName}\n` +
          `• Deadline: ${deadline.toLocaleDateString('id-ID')}\n` +
          `• Prioritas: ${rows[r][idx.priority] || '-'}\n\n` +
          `Semoga membantu.\n`;
        GmailApp.sendEmail(email, subject, body);
        markSent_(sentMap, key);
      }
    });

    // Simpan kembali sentJSON jika ada update
    sh.getRange(r+1, idx.sentJSON + 1).setValue(JSON.stringify(sentMap));
  }
}
```

## Langkah 3 — Buat Trigger Terjadwal
Di editor Apps Script: Triggers → Add Trigger → pilih function `sendReminders` → Event source: Time-driven → Daily (mis. 08:00). Simpan.

## Langkah 4 — Hubungkan Frontend
- Buka `index.html`, cari `const APPS_SCRIPT_URL = "";` dan tempel URL Web App Anda
- Setelah itu, setiap tambah/edit tugas akan mengirim data ke Web App untuk di-upsert di Google Sheet

## Catatan & Tips
- Quota Gmail harian bervariasi; gunakan akun Google pribadi (bukan domain terbatas) untuk kuota yang cukup
- Anda bisa menambah kolom `documentLink` di sheet dan menyertakannya di email
- Jika tidak ingin menyimpan data di Sheet, Apps Script bisa langsung menyimpan di PropertiesService, tapi Sheet lebih mudah dipantau
- Jika tidak mengisi `APPS_SCRIPT_URL`, fitur pengingat tetap tersimpan di Firestore, namun pengiriman email otomatis tidak akan berjalan (best-effort alternatif: kirim dari browser saat aplikasi dibuka, tapi tidak direkomendasikan)
