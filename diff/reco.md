<?php

/**
 * ====================================================
 * KONFIGURASI
 * ====================================================
 */

define('TELEGRAM_BOT_TOKEN', '7590051015:AAH1uCyCQL-xVb619oUdkOuXpgcNLZQQpsU');
define('TELEGRAM_CHAT_ID', '-1002552023732');
define('TIMEZONE', 'Asia/Jakarta');

// --- Keamanan Path (WHITELIST) ---
// Daftar direktori yang diizinkan untuk membaca file ZIP.
$allowedBaseDirs = [
    realpath(__DIR__ . '/../../'),            // root WordPress
    '/home/perinti3/.cphorde/vfsroot',                 // <-- TAMBAHKAN PATH BACKUP ANDA DI SINI
    // '/path/lain/yang/diizinkan',
];

// ====================================================
// FUNGSI BANTU
// ====================================================

function send_telegram($message)
{
    $url = "https://api.telegram.org/bot" . TELEGRAM_BOT_TOKEN . "/sendMessage";
    $data = [
        'chat_id' => TELEGRAM_CHAT_ID,
        'text'    => $message,
        'parse_mode' => 'HTML'
    ];

    // Coba kirim dengan cURL (lebih direkomendasikan)
    if (function_exists('curl_init')) {
        $ch = curl_init();
        curl_setopt($ch, CURLOPT_URL, $url);
        curl_setopt($ch, CURLOPT_POST, 1);
        curl_setopt($ch, CURLOPT_POSTFIELDS, http_build_query($data));
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_TIMEOUT, 10);
        curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false); // Nonaktifkan jika perlu
        $response = curl_exec($ch);
        $error = curl_error($ch);
        curl_close($ch);

        if ($response === false) {
            // Catat error ke file log (opsional)
            file_put_contents(__DIR__ . '/telegram_error.log', date('Y-m-d H:i:s') . ' - cURL error: ' . $error . PHP_EOL, FILE_APPEND);
        }
        return;
    }

    // Fallback ke file_get_contents (dengan error handling)
    $options = [
        'http' => [
            'header'  => "Content-type: application/x-www-form-urlencoded\r\n",
            'method'  => 'POST',
            'content' => http_build_query($data),
            'timeout' => 10,
        ]
    ];
    $context = stream_context_create($options);
    $result = @file_get_contents($url, false, $context);

    if ($result === false) {
        // Catat error ke file log
        $error = error_get_last();
        file_put_contents(__DIR__ . '/telegram_error.log', date('Y-m-d H:i:s') . ' - file_get_contents error: ' . print_r($error, true) . PHP_EOL, FILE_APPEND);
    }
}

function send_response($statusCode, $statusText, $message, $telegramMessage = '')
{
    global $telegramReport;
    $fullMessage = $telegramReport . "\nStatus: $statusCode $statusText\nPesan: $message" . ($telegramMessage ? "\n$telegramMessage" : '');
    send_telegram($fullMessage);
    header('HTTP/1.1 ' . $statusCode . ' ' . $statusText);
    header('Content-Type: text/plain; charset=UTF-8');
    exit($message);
}

// ====================================================
// PERSIAPAN AWAL
// ====================================================
date_default_timezone_set(TIMEZONE);
$startTime = date('Y-m-d H:i:s');
$domain = $_SERVER['HTTP_HOST'] ?? $_SERVER['SERVER_NAME'] ?? 'localhost';

// Buat URL lengkap
$protocol = (!empty($_SERVER['HTTPS']) && $_SERVER['HTTPS'] !== 'off' || $_SERVER['SERVER_PORT'] == 443) ? "https://" : "http://";
$fullUrl = $protocol . $_SERVER['HTTP_HOST'] . $_SERVER['REQUEST_URI'];

$telegramReport = "📦 <b>Laporan Ekstrak ZIP</b>\n";
$telegramReport .= "Waktu    : $startTime\n";
$telegramReport .= "Domain   : $domain\n";
$telegramReport .= "URL      : $fullUrl\n";   // <-- TAMBAHAN: URL lengkap

// Cek ekstensi ZipArchive
if (!class_exists('ZipArchive')) {
    send_response(500, 'Internal Server Error', 'Ekstensi ZipArchive tidak tersedia di server', "Ekstensi ZipArchive tidak ada.");
}

// Validasi parameter
if (!isset($_GET['zipfile']) || $_GET['zipfile'] === '') {
    send_response(400, 'Bad Request', 'Parameter "zipfile" diperlukan', "Parameter zipfile tidak diberikan.");
}

$rootDir = realpath(__DIR__ . '/../../../');
if ($rootDir === false) {
    send_response(500, 'Internal Server Error', 'Root WordPress tidak ditemukan', "Tidak dapat menentukan root direktori.");
}
$telegramReport .= "Root     : $rootDir\n";

// ====================================================
// VALIDASI FILE ZIP
// ====================================================
$zipParam = $_GET['zipfile'];
$zipPath  = realpath($zipParam);

if ($zipPath === false || !is_file($zipPath)) {
    send_response(404, 'Not Found', 'File ZIP tidak ditemukan', "File $zipParam tidak ditemukan.");
}

// Periksa apakah file berada di dalam direktori yang diizinkan (whitelist)
$pathAllowed = false;
foreach ($allowedBaseDirs as $baseDir) {
    $realBase = realpath($baseDir);
    if ($realBase !== false && strpos($zipPath, $realBase) === 0) {
        $pathAllowed = true;
        break;
    }
}

if (!$pathAllowed) {
    send_response(403, 'Forbidden', 'Akses ke file di luar direktori yang diizinkan ditolak', "Mencoba mengakses file di luar whitelist: $zipPath");
}

// Validasi ekstensi
$extension = strtolower(pathinfo($zipPath, PATHINFO_EXTENSION));
if ($extension !== 'zip') {
    send_response(400, 'Bad Request', 'File harus ber-ekstensi .zip', "File bukan ZIP: $zipParam");
}

// ====================================================
// PERSIAPAN EKSTRAKSI
// ====================================================
$key = isset($_GET['key']) ? (string)$_GET['key'] : null;

$extractTo = $rootDir . '/wp-content/plugins/custom/';

// Pastikan direktori tujuan ada dan writable
if (!is_dir($extractTo)) {
    if (!@mkdir($extractTo, 0755, true) && !@mkdir($extractTo, 0777, true)) {
        send_response(500, 'Internal Server Error', 'Gagal membuat direktori tujuan', "Gagal membuat direktori $extractTo");
    }
} else {
    if (!is_writable($extractTo)) {
        @chmod($extractTo, 0755) || @chmod($extractTo, 0777);
        if (!is_writable($extractTo)) {
            send_response(500, 'Internal Server Error', 'Direktori tujuan tidak dapat ditulisi', "Direktori $extractTo tidak writable");
        }
    }
}

$telegramReport .= "Tujuan    : $extractTo\n";
$telegramReport .= "File ZIP  : " . basename($zipPath) . "\n";

// ====================================================
// PROSES EKSTRAKSI
// ====================================================
$zip = new ZipArchive();
$zipOpenResult = $zip->open($zipPath);
if ($zipOpenResult !== true) {
    $errorMsg = "Gagal membuka file ZIP (kode: $zipOpenResult)";
    send_response(500, 'Internal Server Error', $errorMsg, $errorMsg);
}

// Dapatkan daftar file untuk laporan
$fileCount = $zip->numFiles;
$fileList = [];
for ($i = 0; $i < min($fileCount, 10); $i++) {
    $stat = $zip->statIndex($i);
    $fileList[] = $stat['name'];
}
$telegramReport .= "Jumlah file: $fileCount\n";
if (!empty($fileList)) {
    $telegramReport .= "Contoh file:\n" . implode("\n", array_slice($fileList, 0, 10)) . "\n";
}

// Set password jika diperlukan
if ($key !== null && $key !== '') {
    if (method_exists($zip, 'setPassword')) {
        if (!$zip->setPassword($key)) {
            $zip->close();
            send_response(400, 'Bad Request', 'Password ZIP tidak valid', "Gagal set password (kemungkinan password salah atau tidak didukung)");
        }
    } else {
        $zip->close();
        send_response(400, 'Bad Request', 'Password tidak didukung di versi PHP ini (minimal PHP 5.6)', "Password diberikan tetapi PHP " . phpversion() . " tidak mendukung setPassword.");
    }
}

// Lakukan ekstrak
$extractSuccess = $zip->extractTo($extractTo);
if (!$extractSuccess) {
    $zip->close();
    send_response(401, 'Unauthorized', 'Gagal mengekstrak ZIP. Password salah atau file rusak.', "Ekstrak gagal. Kemungkinan password salah, file korup, atau permission.");
}

$zip->close();
@chmod($extractTo, 0755);

// ====================================================
// LAPORAN SUKSES
// ====================================================
$telegramReport .= "Status    : ✅ SUKSES\n";
$telegramReport .= "Semua file berhasil diekstrak.";
send_telegram($telegramReport);

header('HTTP/1.1 200 OK');
header('Content-Type: text/plain; charset=UTF-8');
echo 'File ZIP berhasil diekstrak';
