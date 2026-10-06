<script>
  import { onMount } from 'svelte';

  // State utama transaksi
  const defaultTransactions = [
    { id: 1, tanggal: '2026-09-01', tipe: 'masuk', kategori: 'Kas Masuk', keterangan: 'Saldo Awal Pembukuan Kas Welding 2', nominal: 1850000 },
    { id: 2, tanggal: '2026-09-03', tipe: 'masuk', kategori: 'Infaq Rutin', keterangan: 'Infaq Rutin Jamaah Welding Shift Pagi', nominal: 320000 },
    { id: 3, tanggal: '2026-09-05', tipe: 'keluar', kategori: 'Konsumsi', keterangan: 'Konsumsi Rapat Mingguan & Kopi Tim Welding', nominal: 145000 },
    { id: 4, tanggal: '2026-09-08', tipe: 'masuk', kategori: 'Donatur', keterangan: 'Sumbangan Hamba Allah untuk Operasional', nominal: 500000 },
    { id: 5, tanggal: '2026-09-12', tipe: 'keluar', kategori: 'Operasional', keterangan: 'Pembelian Gas CO2 & Kawat Las Tambahan', nominal: 420000 },
    { id: 6, tanggal: '2026-09-15', tipe: 'masuk', kategori: 'Infaq Rutin', keterangan: 'Infaq Rutin Pertengahan Bulan Welding 2', nominal: 410000 },
    { id: 7, tanggal: '2026-09-18', tipe: 'keluar', kategori: 'Bisyaroh', keterangan: 'Bisyaroh Pengisi Ta\'lim Rutin', nominal: 300000 },
    { id: 8, tanggal: '2026-09-22', tipe: 'keluar', kategori: 'Maintenance', keterangan: 'Service & Penggantian Filter Mesin Las', nominal: 275000 },
    { id: 9, tanggal: '2026-09-26', tipe: 'masuk', kategori: 'Donatur', keterangan: 'Infaq Sukarela Anggota Line Welding B', nominal: 250000 },
    { id: 10, tanggal: '2026-09-29', tipe: 'keluar', kategori: 'Konsumsi', keterangan: 'Snack & Minuman Penutupan Bulan', nominal: 110000 },
    { id: 11, tanggal: '2026-10-02', tipe: 'masuk', kategori: 'Infaq Rutin', keterangan: 'Infaq Awal Bulan Oktober Anggota Welding', nominal: 480000 },
    { id: 12, tanggal: '2026-10-05', tipe: 'keluar', kategori: 'Operasional', keterangan: 'Perlengkapan APD & Sarung Tangan Las', nominal: 185000 }
  ];

  let transactions = $state([...defaultTransactions]);
  let searchQuery = $state('');
  let filterMonth = $state('all');
  let filterType = $state('all');
  let showModal = $state(false);
  let toastMessage = $state('');
  let toastType = $state('success');
  let toastVisible = $state(false);

  // Form State untuk Tambah Transaksi
  let formTanggal = $state(new Date().toISOString().split('T')[0]);
  let formTipe = $state('masuk');
  let formKategori = $state('Infaq Rutin');
  let formKeterangan = $state('');
  let formNominal = $state('');

  // Info Cetak / Penandatangan
  let nomorDokumen = $state('WLD/KEU/2026/IX-01');
  let namaKetua = $state('H. Ahmad Syarifuddin');
  let namaBendahara = $state('Muhammad Ridhoku');
  let kotaCetak = $state('Bekasi');

  onMount(() => {
    const saved = localStorage.getItem('laporan_keuangan_welding_data');
    if (saved) {
      try {
        const parsed = JSON.parse(saved);
        if (Array.isArray(parsed) && parsed.length > 0) {
          transactions = parsed;
        }
      } catch (err) {
        console.error('Failed to parse saved data', err);
      }
    }
  });

  function saveData() {
    localStorage.setItem('laporan_keuangan_welding_data', JSON.stringify(transactions));
  }

  function showToast(msg, type = 'success') {
    toastMessage = msg;
    toastType = type;
    toastVisible = true;
    setTimeout(() => {
      toastVisible = false;
    }, 3000);
  }

  // Format Angka ke Rupiah
  function formatRp(val) {
    return new Intl.NumberFormat('id-ID', {
      style: 'currency',
      currency: 'IDR',
      minimumFractionDigits: 0,
      maximumFractionDigits: 0
    }).format(val || 0);
  }

  // Format Tanggal Indonesia
  function formatTanggalIndo(dateStr) {
    if (!dateStr) return '-';
    try {
      const parts = dateStr.split('-');
      if (parts.length === 3) {
        const d = new Date(Number(parts[0]), Number(parts[1]) - 1, Number(parts[2]));
        return d.toLocaleDateString('id-ID', { day: '2-digit', month: 'long', year: 'numeric' });
      }
      return dateStr;
    } catch {
      return dateStr;
    }
  }

  const tanggalHariIni = new Date().toLocaleDateString('id-ID', {
    day: 'numeric',
    month: 'long',
    year: 'numeric'
  });

  // Ambil list bulan unik dari data
  let availableMonths = $derived.by(() => {
    const set = new Set();
    transactions.forEach(t => {
      if (t.tanggal && t.tanggal.length >= 7) {
        set.add(t.tanggal.substring(0, 7));
      }
    });
    return Array.from(set).sort().reverse();
  });

  // Filter Data
  let filteredTransactions = $derived.by(() => {
    return transactions.filter(t => {
      const matchSearch = (t.keterangan || '').toLowerCase().includes(searchQuery.toLowerCase()) ||
                          (t.kategori || '').toLowerCase().includes(searchQuery.toLowerCase());
      const matchMonth = filterMonth === 'all' || (t.tanggal && t.tanggal.startsWith(filterMonth));
      const matchType = filterType === 'all' || t.tipe === filterType;
      return matchSearch && matchMonth && matchType;
    });
  });

  // Urutkan berdasarkan tanggal & hitung Running Balance
  let processedData = $derived.by(() => {
    const sorted = [...filteredTransactions].sort((a, b) => new Date(a.tanggal).getTime() - new Date(b.tanggal).getTime());
    let currentBalance = 0;
    return sorted.map((item, index) => {
      const masuk = item.tipe === 'masuk' ? Number(item.nominal) : 0;
      const keluar = item.tipe === 'keluar' ? Number(item.nominal) : 0;
      currentBalance += (masuk - keluar);
      return {
        ...item,
        no: index + 1,
        masuk,
        keluar,
        runningBalance: currentBalance
      };
    });
  });

  // Perhitungan Ringkasan
  let totalMasuk = $derived(processedData.reduce((acc, cur) => acc + cur.masuk, 0));
  let totalKeluar = $derived(processedData.reduce((acc, cur) => acc + cur.keluar, 0));
  let saldoAkhir = $derived(totalMasuk - totalKeluar);

  // Label Periode Laporan
  let labelPeriode = $derived.by(() => {
    if (filterMonth === 'all') return 'Seluruh Periode Pembukuan (Tahun 2026)';
    const [y, m] = filterMonth.split('-');
    const date = new Date(Number(y), Number(m) - 1, 1);
    return date.toLocaleDateString('id-ID', { month: 'long', year: 'numeric' }).toUpperCase();
  });

  function tambahTransaksi(e) {
    e.preventDefault();
    const cleanNominal = Number(String(formNominal).replace(/[^0-9]/g, ''));
    if (!formKeterangan || !cleanNominal) {
      showToast('Keterangan dan nominal wajib diisi!', 'error');
      return;
    }

    const newTx = {
      id: Date.now(),
      tanggal: formTanggal,
      tipe: formTipe,
      kategori: formKategori,
      keterangan: formKeterangan,
      nominal: cleanNominal
    };

    transactions = [...transactions, newTx];
    saveData();
    showToast('Transaksi berhasil ditambahkan!');
    formKeterangan = '';
    formNominal = '';
    showModal = false;
  }

  function hapusTransaksi(id) {
    if (confirm('Apakah Anda yakin ingin menghapus transaksi ini?')) {
      transactions = transactions.filter(t => t.id !== id);
      saveData();
      showToast('Transaksi berhasil dihapus');
    }
  }

  function resetDataDefault() {
    if (confirm('Kembalikan ke data transaksi bawaan? Semua modifikasi akan diganti.')) {
      transactions = [...defaultTransactions];
      saveData();
      showToast('Data dikembalikan ke default');
    }
  }

  function cetakLaporan() {
    window.print();
  }

  function salinFormatWA() {
    let teks = `*LAPORAN REKAPITULASI KEUANGAN WELDING 2*\n`;
    teks += `📅 *Periode:* ${labelPeriode}\n`;
    teks += `━━━━━━━━━━━━━━━━━━━━━\n`;
    teks += `📥 *Total Pemasukan:* ${formatRp(totalMasuk)}\n`;
    teks += `📤 *Total Pengeluaran:* ${formatRp(totalKeluar)}\n`;
    teks += `💰 *Sisa Saldo Kas:* ${formatRp(saldoAkhir)}\n`;
    teks += `📊 *Jumlah Transaksi:* ${processedData.length} baris\n`;
    teks += `━━━━━━━━━━━━━━━━━━━━━\n`;
    teks += `*Catatan Terkini:*\n`;

    const limaTerakhir = [...processedData].reverse().slice(0, 5);
    limaTerakhir.forEach((item, i) => {
      const simbol = item.tipe === 'masuk' ? '🟢 (+)' : '🔴 (-)';
      teks += `${i + 1}. ${simbol} ${item.keterangan}: ${formatRp(item.nominal)} (${formatTanggalIndo(item.tanggal)})\n`;
    });

    teks += `━━━━━━━━━━━━━━━━━━━━━\n`;
    teks += `_Dicetak & Diverifikasi pada: ${tanggalHariIni}_\n`;

    if (navigator.clipboard) {
      navigator.clipboard.writeText(teks).then(() => {
        showToast('Ringkasan format WA berhasil disalin ke clipboard!');
      }).catch(() => {
        showToast('Gagal menyalin ringkasan', 'error');
      });
    }
  }

  function unduhCSV() {
    let csv = 'No,Tanggal,Tipe,Kategori,Keterangan,Penerimaan (Rp),Pengeluaran (Rp),Saldo Berjalan (Rp)\n';
    processedData.forEach(row => {
      const ketClean = `"${row.keterangan.replace(/"/g, '""')}"`;
      csv += `${row.no},${row.tanggal},${row.tipe},${row.kategori},${ketClean},${row.masuk},${row.keluar},${row.runningBalance}\n`;
    });
    csv += `,,,TOTAL AKHIR,,${totalMasuk},${totalKeluar},${saldoAkhir}\n`;

    const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `Laporan_Keuangan_Welding_${filterMonth}_${Date.now()}.csv`;
    a.click();
    URL.revokeObjectURL(url);
    showToast('File CSV berhasil diunduh');
  }
</script>

<main class="app-wrapper">
  <!-- TOAST NOTIFICATION -->
  {#if toastVisible}
    <div class="toast-floating {toastType}">
      <span>{toastType === 'success' ? '✅' : '⚠️'}</span>
      <span>{toastMessage}</span>
    </div>
  {/if}

  <!-- TOOLBAR KONTROL ATAS (HANYA DITAMPILKAN DI LAYAR, SEMBUNYI SAAT CETAK) -->
  <header class="control-panel no-print">
    <div class="panel-inner">
      <div class="branding">
        <div class="badge-icon">📊</div>
        <div>
          <h1 class="system-title">Sistem Laporan Keuangan Welding</h1>
          <p class="system-sub">Format Resmi Rekapitulasi Cetak A4 Standard</p>
        </div>
      </div>

      <div class="actions-group">
        <button class="btn btn-primary" onclick={cetakLaporan} title="Cetak atau Simpan PDF (A4)">
          🖨️ Cetak / PDF (A4)
        </button>
        <button class="btn btn-secondary" onclick={() => showModal = true} title="Tambah Data Transaksi">
          ➕ Tambah Transaksi
        </button>
        <button class="btn btn-outline" onclick={salinFormatWA} title="Salin Ringkasan ke WhatsApp">
          📱 Salin WA
        </button>
        <button class="btn btn-outline" onclick={unduhCSV} title="Unduh Spreadsheet CSV">
          📥 Unduh CSV
        </button>
        <button class="btn btn-ghost" onclick={resetDataDefault} title="Kembalikan Contoh Bawaan">
          ↺ Reset Default
        </button>
      </div>
    </div>

    <!-- FILTER BAR -->
    <div class="filter-bar">
      <div class="filter-item">
        <label for="search-input">Cari Transaksi:</label>
        <div class="search-input-wrapper">
          <input
            id="search-input"
            type="text"
            placeholder="Cari keterangan / kategori..."
            bind:value={searchQuery}
          />
          {#if searchQuery}
            <button class="clear-btn" onclick={() => searchQuery = ''}>×</button>
          {/if}
        </div>
      </div>

      <div class="filter-item">
        <label for="filter-month">Periode Bulan:</label>
        <select id="filter-month" bind:value={filterMonth}>
          <option value="all">📅 Seluruh Bulan</option>
          {#each availableMonths as m}
            <option value={m}>Bulan {m}</option>
          {/each}
        </select>
      </div>

      <div class="filter-item">
        <label for="filter-type">Jenis Transaksi:</label>
        <select id="filter-type" bind:value={filterType}>
          <option value="all">Semua Arus Kas</option>
          <option value="masuk">📥 Penerimaan (Masuk)</option>
          <option value="keluar">📤 Pengeluaran (Keluar)</option>
        </select>
      </div>

      <div class="data-counter">
        <span>Menampilkan <strong>{processedData.length}</strong> transaksi</span>
      </div>
    </div>
  </header>

  <!-- ==============================================================
       DOKUMEN FORMAT CETAK A4 (PAGE SHEET)
       ============================================================== -->
  <div class="print-container">
    <div class="a4-page">
      <!-- KOP SURAT RESMI -->
      <header class="official-header">
        <div class="kop-logo-container">
          <div class="kop-emblem">⚙️</div>
        </div>
        <div class="kop-text">
          <h2 class="instansi-name">MAJELIS SHOLAWAT & UNIT SOSIAL WELDING 2</h2>
          <p class="instansi-address">Kawasan Industri & Fabrikasi Komponen • Workshop Welding Division 2 • Bekasi Jawa Barat</p>
          <p class="instansi-contact">Email: kas.welding2@gmail.com • Layanan Informasi & Konfirmasi Pengurus</p>
        </div>
      </header>

      <div class="kop-divider"></div>

      <!-- JUDUL LAPORAN -->
      <section class="report-title-section">
        <h3 class="report-main-title">LAPORAN REKAPITULASI ARUS KAS KEUANGAN</h3>
        <p class="report-period-text">PERIODE: {labelPeriode}</p>
        <div class="report-meta-grid">
          <div class="meta-item"><span>No. Dokumen:</span> <strong>{nomorDokumen}</strong></div>
          <div class="meta-item"><span>Status Buku:</span> <strong>TERVERIFIKASI & SEIMBANG</strong></div>
          <div class="meta-item"><span>Tanggal Terbit:</span> <strong>{tanggalHariIni}</strong></div>
        </div>
      </section>

      <!-- RINGKASAN KEUANGAN (EXECUTIVE SUMMARY CARDS) -->
      <section class="summary-cards-grid">
        <div class="summary-card card-income">
          <div class="card-label">TOTAL PENERIMAAN (DEBIT)</div>
          <div class="card-value">{formatRp(totalMasuk)}</div>
          <div class="card-caption">Akumulasi seluruh kas masuk</div>
        </div>
        <div class="summary-card card-expense">
          <div class="card-label">TOTAL PENGELUARAN (KREDIT)</div>
          <div class="card-value">{formatRp(totalKeluar)}</div>
          <div class="card-caption">Akumulasi operasional & biaya</div>
        </div>
        <div class="summary-card card-balance">
          <div class="card-label">SISA SALDO KAS BERJALAN</div>
          <div class="card-value">{formatRp(saldoAkhir)}</div>
          <div class="card-caption">Posisi saldo kas akhir periode</div>
        </div>
      </section>

      <!-- TABEL REKAPITULASI DETAIL -->
      <section class="table-section">
        <table class="report-table">
          <thead>
            <tr>
              <th class="col-no">NO</th>
              <th class="col-date">TANGGAL</th>
              <th class="col-cat">KATEGORI</th>
              <th class="col-desc">KETERANGAN TRANSAKSI</th>
              <th class="col-num text-right">PENERIMAAN</th>
              <th class="col-num text-right">PENGELUARAN</th>
              <th class="col-num text-right">SALDO AKHIR</th>
              <th class="col-action no-print">AKSI</th>
            </tr>
          </thead>
          <tbody>
            {#if processedData.length === 0}
              <tr>
                <td colspan="8" class="text-center py-4">
                  <em>Tidak ada data transaksi yang sesuai dengan filter.</em>
                </td>
              </tr>
            {:else}
              {#each processedData as item}
                <tr>
                  <td class="text-center font-mono">{item.no}</td>
                  <td class="text-center font-mono">{formatTanggalIndo(item.tanggal)}</td>
                  <td>
                    <span class="badge-cat {item.tipe}">{item.kategori}</span>
                  </td>
                  <td class="desc-cell">{item.keterangan}</td>
                  <td class="text-right font-mono val-masuk">
                    {item.masuk > 0 ? formatRp(item.masuk) : '-'}
                  </td>
                  <td class="text-right font-mono val-keluar">
                    {item.keluar > 0 ? formatRp(item.keluar) : '-'}
                  </td>
                  <td class="text-right font-mono font-bold">
                    {formatRp(item.runningBalance)}
                  </td>
                  <td class="text-center no-print col-action">
                    <button class="btn-del" onclick={() => hapusTransaksi(item.id)} title="Hapus Transaksi">
                      🗑️
                    </button>
                  </td>
                </tr>
              {/each}
            {/if}
          </tbody>
          <tfoot>
            <tr class="total-row">
              <td colspan="4" class="text-right font-bold uppercase">TOTAL KESELURUHAN PERIODE:</td>
              <td class="text-right font-mono font-bold val-masuk">{formatRp(totalMasuk)}</td>
              <td class="text-right font-mono font-bold val-keluar">{formatRp(totalKeluar)}</td>
              <td class="text-right font-mono font-bold val-balance">{formatRp(saldoAkhir)}</td>
              <td class="no-print"></td>
            </tr>
          </tfoot>
        </table>
      </section>

      <!-- CATATAN DOKUMEN -->
      <section class="audit-notes">
        <h4>Catatan & Ketentuan Administrasi:</h4>
        <ol>
          <li>Laporan ini disusun secara berkala dan sah sebagai bukti rekapitulasi arus kas internal Divisi & Majelis Sholawat Welding 2.</li>
          <li>Setiap penerimaan dan pengeluaran didukung oleh bukti kwitansi, nota fisik, serta pencatatan digital bendahara.</li>
          <li>Saldo akhir yang tercantum telah diaudit bersama dan dinyatakan valid untuk dipertanggungjawabkan kepada seluruh anggota.</li>
        </ol>
      </section>

      <!-- KOLOM PENGESAHAN & TANDA TANGAN (SIGNATURE SECTION) -->
      <footer class="signature-section">
        <div class="sig-box">
          <p class="sig-role">Mengetahui & Menyetujui,</p>
          <p class="sig-title">Ketua Majelis Sholawat Welding 2</p>
          <div class="sig-space"></div>
          <p class="sig-name">( {namaKetua} )</p>
          <p class="sig-nip">NIP/ID: WLD-2024-001</p>
        </div>

        <div class="sig-box">
          <p class="sig-role">{kotaCetak}, {tanggalHariIni}</p>
          <p class="sig-title">Bendahara Pengelola Kas</p>
          <div class="sig-space"></div>
          <p class="sig-name">( {namaBendahara} )</p>
          <p class="sig-nip">NIP/ID: WLD-2024-018</p>
        </div>
      </footer>

      <!-- FOOTER DOKUMEN HALAMAN A4 -->
      <div class="print-page-footer">
        <span>Laporan Resmi Rekapitulasi Kas Keuangan Welding 2 • Lembar Dokumen Cetak Standar A4</span>
        <span>Halaman 1 dari 1</span>
      </div>
    </div>
  </div>

  <!-- MODAL TAMBAH TRANSAKSI (NO-PRINT) -->
  {#if showModal}
    <!-- svelte-ignore a11y_click_events_have_key_events a11y_no_noninteractive_element_interactions -->
    <div class="modal-backdrop no-print" onclick={() => showModal = false} role="presentation">
      <div class="modal-card" onclick={(e) => e.stopPropagation()} role="dialog" aria-modal="true" tabindex="-1">
        <div class="modal-header">
          <h3>✍️ Tambah Transaksi Keuangan Baru</h3>
          <button class="modal-close" onclick={() => showModal = false}>✕</button>
        </div>
        <form onsubmit={tambahTransaksi} class="modal-form">
          <div class="form-group">
            <label for="form-tanggal">Tanggal Transaksi:</label>
            <input id="form-tanggal" type="date" bind:value={formTanggal} required />
          </div>

          <div class="form-group">
            <label for="form-tipe">Jenis Arus Kas:</label>
            <select id="form-tipe" bind:value={formTipe}>
              <option value="masuk">📥 Penerimaan / Kas Masuk</option>
              <option value="keluar">📤 Pengeluaran / Kas Keluar</option>
            </select>
          </div>

          <div class="form-group">
            <label for="form-kategori">Kategori:</label>
            <select id="form-kategori" bind:value={formKategori}>
              {#if formTipe === 'masuk'}
                <option value="Infaq Rutin">Infaq Rutin Sholawat</option>
                <option value="Donatur">Donatur / Hamba Allah</option>
                <option value="Kas Masuk">Kas Masuk Line Welding</option>
                <option value="Penerimaan Lain">Penerimaan Lain-Lain</option>
              {:else}
                <option value="Konsumsi">Konsumsi Jamaah & Kopi</option>
                <option value="Operasional">Operasional & Kebutuhan Las</option>
                <option value="Bisyaroh">Bisyaroh Ta'lim / Penceramah</option>
                <option value="Maintenance">Service & Perawatan Alat</option>
                <option value="Sosial">Santunan & Bantuan Sosial</option>
              {/if}
            </select>
          </div>

          <div class="form-group">
            <label for="form-keterangan">Keterangan Detail:</label>
            <input
              id="form-keterangan"
              type="text"
              placeholder="Contoh: Beli kawat las 2 roll, Kopi rapat..."
              bind:value={formKeterangan}
              required
            />
          </div>

          <div class="form-group">
            <label for="form-nominal">Nominal Rupiah (Rp):</label>
            <input
              id="form-nominal"
              type="number"
              min="1000"
              step="1000"
              placeholder="Contoh: 150000"
              bind:value={formNominal}
              required
            />
          </div>

          <div class="modal-actions">
            <button type="button" class="btn btn-ghost" onclick={() => showModal = false}>
              Batal
            </button>
            <button type="submit" class="btn btn-primary">
              Simpan Transaksi
            </button>
          </div>
        </form>
      </div>
    </div>
  {/if}
</main>

<style>
  /* ==============================================================
     GAYA UMUM LAYAR (DESKTOP & MOBILE VIEW)
     ============================================================== */
  .app-wrapper {
    min-height: 100vh;
    display: flex;
    flex-direction: column;
    background-color: #e2e8f0;
    color: #0f172a;
    font-size: 13px;
  }

  .toast-floating {
    position: fixed;
    top: 20px;
    right: 20px;
    z-index: 9999;
    background: #0f172a;
    color: #ffffff;
    padding: 12px 20px;
    border-radius: 8px;
    box-shadow: 0 10px 25px rgba(0,0,0,0.25);
    display: flex;
    align-items: center;
    gap: 10px;
    font-size: 13px;
    font-weight: 600;
  }
  .toast-floating.error {
    background: #b91c1c;
  }

  /* TOOLBAR KONTROL ATAS */
  .control-panel {
    background: #ffffff;
    border-bottom: 2px solid #cbd5e1;
    padding: 16px 24px;
    box-shadow: 0 4px 15px rgba(0,0,0,0.06);
    position: sticky;
    top: 0;
    z-index: 100;
  }

  .panel-inner {
    max-width: 1200px;
    margin: 0 auto;
    display: flex;
    justify-content: space-between;
    align-items: center;
    flex-wrap: wrap;
    gap: 16px;
  }

  .branding {
    display: flex;
    align-items: center;
    gap: 12px;
  }

  .badge-icon {
    font-size: 28px;
    background: #f1f5f9;
    padding: 8px;
    border-radius: 10px;
    border: 1px solid #cbd5e1;
  }

  .system-title {
    font-size: 18px;
    font-weight: 800;
    color: #0f172a;
    margin: 0;
  }

  .system-sub {
    font-size: 12px;
    color: #64748b;
    margin: 0;
  }

  .actions-group {
    display: flex;
    align-items: center;
    gap: 8px;
    flex-wrap: wrap;
  }

  .btn {
    padding: 9px 16px;
    font-size: 12.5px;
    font-weight: 700;
    border-radius: 7px;
    cursor: pointer;
    transition: all 0.15s ease-in-out;
    border: none;
    display: inline-flex;
    align-items: center;
    gap: 6px;
  }

  .btn-primary {
    background-color: #047857;
    color: #ffffff;
    box-shadow: 0 2px 8px rgba(4, 120, 87, 0.3);
  }
  .btn-primary:hover {
    background-color: #065f46;
  }

  .btn-secondary {
    background-color: #1e293b;
    color: #ffffff;
  }
  .btn-secondary:hover {
    background-color: #0f172a;
  }

  .btn-outline {
    background: transparent;
    border: 1px solid #cbd5e1;
    color: #334155;
  }
  .btn-outline:hover {
    background: #f8fafc;
    border-color: #94a3b8;
  }

  .btn-ghost {
    background: transparent;
    color: #64748b;
  }
  .btn-ghost:hover {
    background: #f1f5f9;
    color: #0f172a;
  }

  .filter-bar {
    max-width: 1200px;
    margin: 14px auto 0 auto;
    padding-top: 12px;
    border-top: 1px solid #e2e8f0;
    display: flex;
    align-items: center;
    gap: 20px;
    flex-wrap: wrap;
  }

  .filter-item {
    display: flex;
    align-items: center;
    gap: 8px;
    font-size: 12px;
    font-weight: 600;
    color: #475569;
  }

  .filter-item select,
  .search-input-wrapper input {
    padding: 6px 10px;
    border: 1px solid #cbd5e1;
    border-radius: 6px;
    font-size: 12px;
    background: #ffffff;
    color: #0f172a;
    outline: none;
  }

  .search-input-wrapper {
    position: relative;
    display: inline-block;
  }

  .search-input-wrapper input {
    width: 220px;
  }

  .clear-btn {
    position: absolute;
    right: 6px;
    top: 50%;
    transform: translateY(-50%);
    background: none;
    border: none;
    color: #94a3b8;
    cursor: pointer;
    font-weight: bold;
  }

  .data-counter {
    margin-left: auto;
    font-size: 12px;
    color: #64748b;
  }

  /* ==============================================================
     CONTAINER & LEMBAR A4 DOKUMEN CETAK
     ============================================================== */
  .print-container {
    padding: 24px 16px 60px 16px;
    display: flex;
    justify-content: center;
    align-items: flex-start;
    overflow-x: auto;
  }

  .a4-page {
    background: #ffffff;
    width: 210mm;
    min-height: 297mm;
    padding: 16mm 18mm 16mm 18mm;
    box-shadow: 0 10px 30px rgba(0, 0, 0, 0.12);
    border: 1px solid #cbd5e1;
    display: flex;
    flex-direction: column;
    box-sizing: border-box;
    position: relative;
  }

  /* KOP SURAT */
  .official-header {
    display: flex;
    align-items: center;
    gap: 16px;
    padding-bottom: 12px;
  }

  .kop-logo-container {
    flex-shrink: 0;
  }

  .kop-emblem {
    width: 58px;
    height: 58px;
    border: 2px solid #047857;
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 30px;
    background: #ecfdf5;
  }

  .kop-text {
    flex: 1;
    text-align: center;
  }

  .instansi-name {
    font-size: 16pt;
    font-weight: 800;
    color: #064e3b;
    letter-spacing: 0.5px;
    margin: 0;
    line-height: 1.2;
  }

  .instansi-address {
    font-size: 8.5pt;
    color: #334155;
    margin: 3px 0 1px 0;
    font-weight: 500;
  }

  .instansi-contact {
    font-size: 8pt;
    color: #64748b;
    margin: 0;
  }

  .kop-divider {
    border-bottom: 3px solid #0f172a;
    border-top: 1px solid #0f172a;
    height: 3px;
    margin-bottom: 14px;
  }

  /* JUDUL LAPORAN */
  .report-title-section {
    text-align: center;
    margin-bottom: 14px;
  }

  .report-main-title {
    font-size: 13pt;
    font-weight: 800;
    color: #0f172a;
    letter-spacing: 1px;
    margin: 0 0 3px 0;
  }

  .report-period-text {
    font-size: 9.5pt;
    font-weight: 700;
    color: #047857;
    margin: 0 0 8px 0;
  }

  .report-meta-grid {
    display: flex;
    justify-content: space-between;
    background: #f8fafc;
    border: 1px solid #e2e8f0;
    border-radius: 5px;
    padding: 6px 12px;
    font-size: 8pt;
    color: #475569;
  }

  /* EXECUTIVE SUMMARY CARDS */
  .summary-cards-grid {
    display: grid;
    grid-template-columns: 1fr 1fr 1fr;
    gap: 12px;
    margin-bottom: 16px;
  }

  .summary-card {
    border-radius: 6px;
    padding: 10px 14px;
    border: 1px solid #cbd5e1;
    background: #ffffff;
  }

  .card-income {
    background: #f0fdf4;
    border-color: #86efac;
  }

  .card-expense {
    background: #fef2f2;
    border-color: #fca5a5;
  }

  .card-balance {
    background: #f8fafc;
    border-color: #94a3b8;
  }

  .card-label {
    font-size: 7.5pt;
    font-weight: 700;
    letter-spacing: 0.5px;
    color: #475569;
    margin-bottom: 2px;
  }

  .card-value {
    font-size: 13pt;
    font-weight: 800;
    line-height: 1.2;
    font-family: ui-monospace, monospace;
  }

  .card-income .card-value { color: #15803d; }
  .card-expense .card-value { color: #b91c1c; }
  .card-balance .card-value { color: #0f172a; }

  .card-caption {
    font-size: 7pt;
    color: #64748b;
    margin-top: 2px;
  }

  /* TABEL DETAIL */
  .table-section {
    margin-bottom: 14px;
    flex: 1;
  }

  .report-table {
    width: 100%;
    border-collapse: collapse;
    font-size: 8pt;
    color: #0f172a;
  }

  .report-table th,
  .report-table td {
    border: 1px solid #cbd5e1;
    padding: 5px 7px;
  }

  .report-table thead th {
    background: #0f172a;
    color: #ffffff;
    font-weight: 700;
    font-size: 7.5pt;
    letter-spacing: 0.3px;
    text-align: center;
  }

  .report-table tbody tr:nth-child(even) {
    background-color: #f8fafc;
  }

  .col-no { width: 4%; }
  .col-date { width: 14%; }
  .col-cat { width: 14%; }
  .col-desc { width: 32%; }
  .col-num { width: 12%; }
  .col-action { width: 4%; }

  .desc-cell {
    font-weight: 500;
    color: #1e293b;
  }

  .badge-cat {
    display: inline-block;
    padding: 1px 6px;
    border-radius: 4px;
    font-size: 7pt;
    font-weight: 600;
  }
  .badge-cat.masuk {
    background: #dcfce7;
    color: #166534;
  }
  .badge-cat.keluar {
    background: #fee2e2;
    color: #991b1b;
  }

  .val-masuk { color: #15803d; }
  .val-keluar { color: #b91c1c; }
  .val-balance { color: #0f172a; }

  .total-row {
    background: #f1f5f9 !important;
    border-top: 2px solid #0f172a;
  }

  .btn-del {
    background: none;
    border: none;
    cursor: pointer;
    font-size: 11px;
    opacity: 0.6;
    transition: opacity 0.15s;
  }
  .btn-del:hover {
    opacity: 1;
  }

  /* CATATAN AUDIT */
  .audit-notes {
    border: 1px dashed #cbd5e1;
    border-radius: 4px;
    padding: 8px 12px;
    background: #fafafa;
    margin-bottom: 18px;
    font-size: 7.5pt;
    color: #475569;
  }

  .audit-notes h4 {
    font-size: 8pt;
    font-weight: 700;
    color: #0f172a;
    margin: 0 0 4px 0;
  }

  .audit-notes ol {
    margin: 0;
    padding-left: 16px;
    line-height: 1.4;
  }

  /* TANDA TANGAN (SIGNATURE SECTION) */
  .signature-section {
    display: flex;
    justify-content: space-between;
    margin-top: 10px;
    padding: 0 20px;
  }

  .sig-box {
    text-align: center;
    width: 220px;
  }

  .sig-role {
    font-size: 8pt;
    color: #64748b;
    margin: 0;
  }

  .sig-title {
    font-size: 8.5pt;
    font-weight: 700;
    color: #0f172a;
    margin: 2px 0 0 0;
  }

  .sig-space {
    height: 52px;
  }

  .sig-name {
    font-size: 9pt;
    font-weight: 800;
    text-decoration: underline;
    color: #0f172a;
    margin: 0;
  }

  .sig-nip {
    font-size: 7.5pt;
    color: #64748b;
    margin: 2px 0 0 0;
  }

  /* FOOTER LEMBAR CETAK */
  .print-page-footer {
    margin-top: 16px;
    padding-top: 6px;
    border-top: 1px solid #e2e8f0;
    display: flex;
    justify-content: space-between;
    font-size: 7pt;
    color: #94a3b8;
  }

  /* MODAL */
  .modal-backdrop {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(15, 23, 42, 0.6);
    backdrop-filter: blur(4px);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 999;
    padding: 16px;
  }

  .modal-card {
    background: #ffffff;
    width: 100%;
    max-width: 460px;
    border-radius: 12px;
    box-shadow: 0 20px 40px rgba(0, 0, 0, 0.2);
    overflow: hidden;
  }

  .modal-header {
    background: #f8fafc;
    border-bottom: 1px solid #e2e8f0;
    padding: 14px 20px;
    display: flex;
    justify-content: space-between;
    align-items: center;
  }

  .modal-header h3 {
    margin: 0;
    font-size: 14px;
    font-weight: 800;
  }

  .modal-close {
    background: none;
    border: none;
    font-size: 16px;
    cursor: pointer;
    color: #64748b;
  }

  .modal-form {
    padding: 20px;
    display: flex;
    flex-direction: column;
    gap: 12px;
  }

  .form-group {
    display: flex;
    flex-direction: column;
    gap: 4px;
  }

  .form-group label {
    font-size: 11.5px;
    font-weight: 700;
    color: #334155;
  }

  .form-group input,
  .form-group select {
    padding: 8px 12px;
    border: 1px solid #cbd5e1;
    border-radius: 6px;
    font-size: 13px;
    outline: none;
  }

  .modal-actions {
    display: flex;
    justify-content: flex-end;
    gap: 10px;
    margin-top: 12px;
  }

  /* HELPER TYPOGRAPHY */
  .text-center { text-align: center; }
  .text-right { text-align: right; }
  .font-mono { font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace; }
  .font-bold { font-weight: 700; }
  .uppercase { text-transform: uppercase; }

  /* ==============================================================
     ATURAN MEDIA CETAK (PRINT & EXPORT A4 PORTRAIT)
     ============================================================== */
  @page {
    size: A4 portrait;
    margin: 10mm 12mm 10mm 12mm;
  }

  @media print {
    *, *::before, *::after {
      box-shadow: none !important;
      text-shadow: none !important;
      -webkit-print-color-adjust: exact !important;
      print-color-adjust: exact !important;
    }

    :global(body) {
      background: #ffffff !important;
      margin: 0 !important;
      padding: 0 !important;
    }

    .app-wrapper {
      background: #ffffff !important;
      padding: 0 !important;
      margin: 0 !important;
      min-height: auto !important;
    }

    .no-print {
      display: none !important;
    }

    .print-container {
      padding: 0 !important;
      margin: 0 !important;
      display: block !important;
      overflow: visible !important;
    }

    .a4-page {
      width: 100% !important;
      min-height: auto !important;
      height: auto !important;
      padding: 0 !important;
      margin: 0 !important;
      border: none !important;
      box-shadow: none !important;
      page-break-inside: avoid !important;
      break-inside: avoid !important;
    }

    .report-table {
      font-size: 7.5pt !important;
    }

    .report-table th,
    .report-table td {
      padding: 4px 5px !important;
    }

    .kop-emblem {
      -webkit-print-color-adjust: exact !important;
      print-color-adjust: exact !important;
    }

    .summary-card {
      -webkit-print-color-adjust: exact !important;
      print-color-adjust: exact !important;
    }

    .signature-section {
      page-break-inside: avoid !important;
      break-inside: avoid !important;
    }
  }

  @media screen and (max-width: 840px) {
    .control-panel {
      padding: 12px 14px;
    }
    .panel-inner {
      flex-direction: column;
      align-items: flex-start;
    }
    .filter-bar {
      flex-direction: column;
      align-items: stretch;
      gap: 10px;
    }
    .search-input-wrapper input {
      width: 100%;
    }
    .data-counter {
      margin-left: 0;
    }
    .print-container {
      padding: 10px 4px;
    }
    .a4-page {
      width: 100%;
      min-height: auto;
      padding: 14px 10px;
    }
    .summary-cards-grid {
      grid-template-columns: 1fr;
    }
    .signature-section {
      flex-direction: column;
      align-items: center;
      gap: 24px;
    }
  }
</style>
