<script>
  import { onMount } from 'svelte';

  // Dataset Transaksi Awal (Realistis Divisi & Kas Welding 2)
  const defaultTransactions = [
    { id: 1, tanggal: '2026-09-01', tipe: 'masuk', kategori: 'Kas Masuk', keterangan: 'Saldo Awal Kas Pembukuan Welding 2', nominal: 1850000 },
    { id: 2, tanggal: '2026-09-03', tipe: 'masuk', kategori: 'Infaq Rutin', keterangan: 'Infaq Rutin Jamaah Welding Shift Pagi', nominal: 320000 },
    { id: 3, tanggal: '2026-09-05', tipe: 'keluar', kategori: 'Konsumsi', keterangan: 'Konsumsi Rapat Mingguan & Kopi Tim Welding', nominal: 145000 },
    { id: 4, tanggal: '2026-09-08', tipe: 'masuk', kategori: 'Donatur', keterangan: 'Sumbangan Hamba Allah untuk Operasional', nominal: 500000 },
    { id: 5, tanggal: '2026-09-12', tipe: 'keluar', kategori: 'Operasional', keterangan: 'Pembelian Gas CO2 & Kawat Las Tambahan', nominal: 420000 },
    { id: 6, tanggal: '2026-09-15', tipe: 'masuk', kategori: 'Infaq Rutin', keterangan: 'Infaq Rutin Pertengahan Bulan Welding 2', nominal: 410000 },
    { id: 7, tanggal: '2026-09-18', tipe: 'keluar', kategori: 'Bisyaroh', keterangan: 'Bisyaroh Pengisi Ta\'lim Rutin Majelis', nominal: 300000 },
    { id: 8, tanggal: '2026-09-22', tipe: 'keluar', kategori: 'Maintenance', keterangan: 'Service & Penggantian Filter Mesin Las', nominal: 275000 },
    { id: 9, tanggal: '2026-09-26', tipe: 'masuk', kategori: 'Donatur', keterangan: 'Infaq Sukarela Anggota Line Welding B', nominal: 250000 },
    { id: 10, tanggal: '2026-09-29', tipe: 'keluar', kategori: 'Konsumsi', keterangan: 'Snack & Minuman Penutupan Bulan', nominal: 110000 },
    { id: 11, tanggal: '2026-10-02', tipe: 'masuk', kategori: 'Infaq Rutin', keterangan: 'Infaq Awal Bulan Oktober Anggota Welding', nominal: 480000 },
    { id: 12, tanggal: '2026-10-05', tipe: 'keluar', kategori: 'Operasional', keterangan: 'Perlengkapan APD & Sarung Tangan Las', nominal: 185000 }
  ];

  // State Reaktif (Svelte 5 Runes)
  let transactions = $state([...defaultTransactions]);
  let currentTab = $state('dashboard'); // 'dashboard' | 'preview'
  let searchQuery = $state('');
  let filterMonth = $state('all');
  let filterType = $state('all'); // 'all' | 'masuk' | 'keluar'
  let showModal = $state(false);

  // Form State Tambah Transaksi
  let formTanggal = $state(new Date().toISOString().split('T')[0]);
  let formTipe = $state('masuk');
  let formKategori = $state('Infaq Rutin');
  let formKeterangan = $state('');
  let formNominal = $state('');

  // Toast Notification
  let toastMsg = $state('');
  let toastType = $state('success');
  let toastActive = $state(false);

  // Info Dokumen Resmi
  let nomorDokumen = $state('WLD/KEU/2026/IX-01');
  let namaKetua = $state('H. Ahmad Syarifuddin');
  let namaBendahara = $state('Muhammad Ridhoku');
  let kotaCetak = $state('Bekasi');

  onMount(() => {
    const saved = localStorage.getItem('laporan_keuangan_welding_v2');
    if (saved) {
      try {
        const parsed = JSON.parse(saved);
        if (Array.isArray(parsed) && parsed.length > 0) {
          transactions = parsed;
        }
      } catch (e) {
        console.error(e);
      }
    }
  });

  function saveData() {
    localStorage.setItem('laporan_keuangan_welding_v2', JSON.stringify(transactions));
  }

  function triggerToast(msg, type = 'success') {
    toastMsg = msg;
    toastType = type;
    toastActive = true;
    setTimeout(() => { toastActive = false; }, 3000);
  }

  // Format Helper
  function formatRp(val) {
    return new Intl.NumberFormat('id-ID', {
      style: 'currency',
      currency: 'IDR',
      minimumFractionDigits: 0
    }).format(val || 0);
  }

  function formatTanggalIndo(dateStr) {
    if (!dateStr) return '-';
    try {
      const p = dateStr.split('-');
      if (p.length === 3) {
        const d = new Date(Number(p[0]), Number(p[1]) - 1, Number(p[2]));
        return d.toLocaleDateString('id-ID', { day: '2-digit', month: 'short', year: 'numeric' });
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

  // Kategori Preset Icon & Color
  const categoryMeta = {
    'Infaq Rutin': { icon: '🕌', color: '#10b981' },
    'Donatur': { icon: '🤲', color: '#059669' },
    'Kas Masuk': { icon: '💼', color: '#3b82f6' },
    'Penerimaan Lain': { icon: '📥', color: '#6366f1' },
    'Konsumsi': { icon: '☕', color: '#f59e0b' },
    'Operasional': { icon: '⚙️', color: '#ef4444' },
    'Bisyaroh': { icon: '👳', color: '#8b5cf6' },
    'Maintenance': { icon: '🛠️', color: '#ec4899' },
    'Sosial': { icon: '🤝', color: '#14b8a6' }
  };

  // List Bulan Unik
  let availableMonths = $derived.by(() => {
    const set = new Set();
    transactions.forEach(t => {
      if (t.tanggal && t.tanggal.length >= 7) set.add(t.tanggal.substring(0, 7));
    });
    return Array.from(set).sort().reverse();
  });

  // Filtered & Sorted Data
  let filteredTransactions = $derived.by(() => {
    return transactions.filter(t => {
      const matchSearch = (t.keterangan || '').toLowerCase().includes(searchQuery.toLowerCase()) ||
                          (t.kategori || '').toLowerCase().includes(searchQuery.toLowerCase());
      const matchMonth = filterMonth === 'all' || (t.tanggal && t.tanggal.startsWith(filterMonth));
      const matchType = filterType === 'all' || t.tipe === filterType;
      return matchSearch && matchMonth && matchType;
    });
  });

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

  // Financial Metrics
  let totalMasuk = $derived(processedData.reduce((acc, cur) => acc + cur.masuk, 0));
  let totalKeluar = $derived(processedData.reduce((acc, cur) => acc + cur.keluar, 0));
  let saldoAkhir = $derived(totalMasuk - totalKeluar);
  let rasioSurplus = $derived(totalMasuk > 0 ? Math.round(((totalMasuk - totalKeluar) / totalMasuk) * 100) : 0);

  // Visual Donut Chart Percentages
  let totalVolume = $derived(totalMasuk + totalKeluar);
  let masukPercent = $derived(totalVolume > 0 ? (totalMasuk / totalVolume) * 100 : 50);
  let keluarPercent = $derived(totalVolume > 0 ? (totalKeluar / totalVolume) * 100 : 50);

  // Label Periode
  let labelPeriode = $derived.by(() => {
    if (filterMonth === 'all') return 'Seluruh Periode Pembukuan (2026)';
    const [y, m] = filterMonth.split('-');
    const date = new Date(Number(y), Number(m) - 1, 1);
    return date.toLocaleDateString('id-ID', { month: 'long', year: 'numeric' }).toUpperCase();
  });

  // Quick Chips Nominal
  function tambahChipNominal(val) {
    const cur = Number(String(formNominal).replace(/[^0-9]/g, '')) || 0;
    formNominal = String(cur + val);
  }

  function submitTambah(e) {
    e.preventDefault();
    const cleanNom = Number(String(formNominal).replace(/[^0-9]/g, ''));
    if (!formKeterangan || !cleanNom) {
      triggerToast('Keterangan dan nominal wajib diisi!', 'error');
      return;
    }

    const newTx = {
      id: Date.now(),
      tanggal: formTanggal,
      tipe: formTipe,
      kategori: formKategori,
      keterangan: formKeterangan,
      nominal: cleanNom
    };

    transactions = [...transactions, newTx];
    saveData();
    triggerToast('Transaksi berhasil ditambahkan!');
    formKeterangan = '';
    formNominal = '';
    showModal = false;
  }

  function hapus(id) {
    transactions = transactions.filter(t => t.id !== id);
    saveData();
    triggerToast('Transaksi dihapus');
  }

  function resetBawaan() {
    transactions = [...defaultTransactions];
    saveData();
    triggerToast('Data disetel ulang ke contoh default');
  }

  function cetakDokumen() {
    window.print();
  }

  function salinWA() {
    let teks = `📊 *LAPORAN REKAPITULASI KEUANGAN WELDING 2*\n`;
    teks += `📅 *Periode:* ${labelPeriode}\n`;
    teks += `━━━━━━━━━━━━━━━━━━━━━\n`;
    teks += `💰 *Saldo Kas Bersih:* ${formatRp(saldoAkhir)}\n`;
    teks += `📥 *Total Penerimaan:* ${formatRp(totalMasuk)}\n`;
    teks += `📤 *Total Pengeluaran:* ${formatRp(totalKeluar)}\n`;
    teks += `📈 *Rasio Surplus:* ${rasioSurplus}%\n`;
    teks += `📋 *Jumlah Transaksi:* ${processedData.length} baris\n`;
    teks += `━━━━━━━━━━━━━━━━━━━━━\n`;
    teks += `*3 Catatan Arus Kas Terakhir:*\n`;

    const recent = [...processedData].reverse().slice(0, 3);
    recent.forEach((t, i) => {
      const mark = t.tipe === 'masuk' ? '🟢 (+)' : '🔴 (-)';
      teks += `${i + 1}. ${mark} ${t.keterangan}: *${formatRp(t.nominal)}* (${formatTanggalIndo(t.tanggal)})\n`;
    });
    teks += `━━━━━━━━━━━━━━━━━━━━━\n`;
    teks += `_Diverifikasi resmi oleh Pengurus Welding 2_\n`;

    if (navigator.clipboard) {
      navigator.clipboard.writeText(teks).then(() => {
        triggerToast('Ringkasan format WA berhasil disalin!');
      }).catch(() => {
        triggerToast('Gagal menyalin ringkasan', 'error');
      });
    }
  }

  function exportCSV() {
    let csv = 'No,Tanggal,Tipe,Kategori,Keterangan,Penerimaan (Rp),Pengeluaran (Rp),Saldo Berjalan (Rp)\n';
    processedData.forEach(r => {
      const cleanKet = `"${r.keterangan.replace(/"/g, '""')}"`;
      csv += `${r.no},${r.tanggal},${r.tipe},${r.kategori},${cleanKet},${r.masuk},${r.keluar},${r.runningBalance}\n`;
    });
    csv += `,,,TOTAL AKHIR,,${totalMasuk},${totalKeluar},${saldoAkhir}\n`;

    const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `Laporan_Keuangan_Welding_${Date.now()}.csv`;
    a.click();
    URL.revokeObjectURL(url);
    triggerToast('Spreadsheet CSV berhasil diunduh');
  }
</script>

<div class="interactive-app">
  <!-- FLOATING TOAST -->
  {#if toastActive}
    <div class="toast-bubble {toastType}">
      <span>{toastType === 'success' ? '✨' : '⚠️'}</span>
      <span>{toastMsg}</span>
    </div>
  {/if}

  <!-- TOP APP NAVIGATION BAR (NO-PRINT) -->
  <nav class="top-navbar no-print">
    <div class="nav-container">
      <div class="brand-group">
        <div class="brand-logo">⚡</div>
        <div>
          <div class="brand-title">WELDING 2 FINANCIAL</div>
          <div class="brand-tag">Sistem Rekapitulasi Kas & Dokumen A4</div>
        </div>
      </div>

      <!-- VIEW TOGGLE TABS -->
      <div class="tab-pill-group">
        <button
          class="tab-pill {currentTab === 'dashboard' ? 'active' : ''}"
          onclick={() => currentTab = 'dashboard'}
        >
          📱 Dashboard Interaktif
        </button>
        <button
          class="tab-pill {currentTab === 'preview' ? 'active' : ''}"
          onclick={() => currentTab = 'preview'}
        >
          📄 Format Cetak A4
        </button>
      </div>

      <!-- TOP ACTIONS -->
      <div class="nav-actions">
        <button class="action-btn btn-print-quick" onclick={cetakDokumen} title="Cetak Lembar A4 Sekarang">
          🖨️ Cetak A4
        </button>
        <button class="action-btn btn-add-quick" onclick={() => showModal = true} title="Tambah Catatan Kas">
          ➕ Catat Kas
        </button>
      </div>
    </div>
  </nav>

  <!-- ==============================================================
       VIEW 1: DASHBOARD INTERAKTIF
       ============================================================== -->
  {#if currentTab === 'dashboard'}
    <main class="dashboard-screen no-print">
      <div class="screen-container">
        <!-- HERO WEALTH & STATS BANNER -->
        <section class="hero-balance-section">
          <!-- CARD 1: SALDO UTAMA (HERO) -->
          <div class="hero-card hero-balance-card">
            <div class="card-glass-glow"></div>
            <div class="hero-card-header">
              <span class="pill-badge-gold">💼 Total Saldo Kas Tersedia</span>
              <span class="live-date-pill">📅 {tanggalHariIni}</span>
            </div>
            <div class="balance-number">{formatRp(saldoAkhir)}</div>
            <div class="hero-card-footer">
              <div class="ratio-badge">
                <span class="ratio-dot"></span>
                <span>Efisiensi Kas: <strong>{rasioSurplus >= 0 ? `+${rasioSurplus}% Surplus` : `${rasioSurplus}% Defisit`}</strong></span>
              </div>
              <div class="action-mini-group">
                <button class="mini-btn" onclick={salinWA} title="Salin Ringkasan ke WhatsApp">📱 Salin WA</button>
                <button class="mini-btn" onclick={exportCSV} title="Unduh File CSV">⬇️ Excel CSV</button>
              </div>
            </div>
          </div>

          <!-- CARD 2: PEMASUKAN -->
          <div class="hero-card income-stat-card">
            <div class="stat-top">
              <div class="stat-icon income-icon">📥</div>
              <span class="stat-trend positive">+ Penerimaan</span>
            </div>
            <div class="stat-title">Total Pemasukan</div>
            <div class="stat-value text-emerald">{formatRp(totalMasuk)}</div>
            <div class="stat-bar-container">
              <div class="stat-bar-fill income-bar" style="width: {masukPercent}%;"></div>
            </div>
            <span class="stat-sub">{masukPercent.toFixed(1)}% dari total arus transaksi</span>
          </div>

          <!-- CARD 3: PENGELUARAN -->
          <div class="hero-card expense-stat-card">
            <div class="stat-top">
              <div class="stat-icon expense-icon">📤</div>
              <span class="stat-trend negative">- Pengeluaran</span>
            </div>
            <div class="stat-title">Total Pengeluaran</div>
            <div class="stat-value text-rose">{formatRp(totalKeluar)}</div>
            <div class="stat-bar-container">
              <div class="stat-bar-fill expense-bar" style="width: {keluarPercent}%;"></div>
            </div>
            <span class="stat-sub">{keluarPercent.toFixed(1)}% dari total arus transaksi</span>
          </div>
        </section>

        <!-- INTERACTIVE CONTROLS & FILTER ROW -->
        <section class="interactive-filter-bar">
          <div class="search-box">
            <span class="search-glyph">🔍</span>
            <input
              type="text"
              placeholder="Cari transaksi, infaq, alat las, kopi..."
              bind:value={searchQuery}
            />
            {#if searchQuery}
              <button class="clear-search" onclick={() => searchQuery = ''}>✕</button>
            {/if}
          </div>

          <!-- FILTER PILLS -->
          <div class="chip-filter-row">
            <button
              class="filter-chip {filterType === 'all' ? 'active' : ''}"
              onclick={() => filterType = 'all'}
            >
              Semua ({processedData.length})
            </button>
            <button
              class="filter-chip chip-in {filterType === 'masuk' ? 'active' : ''}"
              onclick={() => filterType = 'masuk'}
            >
              📥 Pemasukan
            </button>
            <button
              class="filter-chip chip-out {filterType === 'keluar' ? 'active' : ''}"
              onclick={() => filterType = 'keluar'}
            >
              📤 Pengeluaran
            </button>
          </div>

          <!-- MONTH DROPDOWN -->
          <div class="month-select-wrapper">
            <select bind:value={filterMonth} class="styled-select">
              <option value="all">📅 Semua Periode Bulan</option>
              {#each availableMonths as m}
                <option value={m}>Bulan {m}</option>
              {/each}
            </select>
          </div>

          <div class="quick-view-switch">
            <button class="btn-preview-switch" onclick={() => currentTab = 'preview'}>
              Lihat Lembar Cetak A4 ➔
            </button>
          </div>
        </section>

        <!-- INTERACTIVE TRANSACTIONS DATA GRID -->
        <section class="transactions-view">
          <div class="view-header">
            <div>
              <h2 class="view-title">Daftar Rekapitulasi Kas</h2>
              <p class="view-subtitle">Transaksi terlacak pada periode ini: {processedData.length} transaksi</p>
            </div>
            <div class="header-tools">
              <button class="tool-btn" onclick={resetBawaan} title="Kembalikan Contoh Bawaan">
                ↺ Muat Ulang Data
              </button>
            </div>
          </div>

          <div class="table-card-wrapper">
            <table class="interactive-table">
              <thead>
                <tr>
                  <th style="width: 50px;">NO</th>
                  <th style="width: 130px;">TANGGAL</th>
                  <th style="width: 170px;">KATEGORI</th>
                  <th>KETERANGAN TRANSAKSI</th>
                  <th style="width: 150px;" class="text-right">NOMINAL</th>
                  <th style="width: 160px;" class="text-right">SALDO AKHIR</th>
                  <th style="width: 60px;" class="text-center">AKSI</th>
                </tr>
              </thead>
              <tbody>
                {#if processedData.length === 0}
                  <tr>
                    <td colspan="7" class="empty-state">
                      <div class="empty-emoji">🔍</div>
                      <p>Tidak ada transaksi yang cocok dengan filter atau kata kunci pencarian.</p>
                      <button class="btn-clear-filter" onclick={() => { searchQuery = ''; filterType = 'all'; filterMonth = 'all'; }}>
                        Hapus Semua Filter
                      </button>
                    </td>
                  </tr>
                {:else}
                  {#each processedData as item}
                    <tr class="tx-row {item.tipe}">
                      <td class="text-center font-mono num-sub">{item.no}</td>
                      <td class="font-mono text-date">{formatTanggalIndo(item.tanggal)}</td>
                      <td>
                        <span class="cat-pill {item.tipe}">
                          <span class="cat-icon">{categoryMeta[item.kategori]?.icon || '📌'}</span>
                          <span>{item.kategori}</span>
                        </span>
                      </td>
                      <td class="desc-cell font-medium">{item.keterangan}</td>
                      <td class="text-right font-mono font-bold {item.tipe === 'masuk' ? 'val-in' : 'val-out'}">
                        {item.tipe === 'masuk' ? `+ ${formatRp(item.masuk)}` : `- ${formatRp(item.keluar)}`}
                      </td>
                      <td class="text-right font-mono font-bold val-running">
                        {formatRp(item.runningBalance)}
                      </td>
                      <td class="text-center">
                        <button class="action-del-btn" onclick={() => hapus(item.id)} title="Hapus Transaksi">
                          🗑️
                        </button>
                      </td>
                    </tr>
                  {/each}
                {/if}
              </tbody>
            </table>
          </div>
        </section>
      </div>
    </main>
  {/if}

  <!-- ==============================================================
       VIEW 2: PRATINJAU & LEMBAR CETAK DOKUMEN RESMI A4
       (Tetap aktif dan dicetak saat @media print)
       ============================================================== -->
  <section class="print-document-screen {currentTab !== 'preview' ? 'hidden-on-screen' : ''}">
    <!-- PREVIEW ACTION BAR (NO-PRINT) -->
    <div class="preview-toolbar no-print">
      <div class="toolbar-content">
        <div class="toolbar-info">
          <span class="paper-badge">📄 FORMAT CETAK A4</span>
          <span>Dokumen standar resmi siap dicetak atau disimpan sebagai PDF.</span>
        </div>
        <div class="toolbar-btns">
          <button class="btn-action-primary" onclick={cetakDokumen}>
            🖨️ Cetak Halaman Ini
          </button>
          <button class="btn-action-secondary" onclick={() => currentTab = 'dashboard'}>
            📱 Kembali ke Dashboard
          </button>
        </div>
      </div>
    </div>

    <!-- SHEET A4 CONTAINER -->
    <div class="a4-sheet-wrapper">
      <article class="a4-paper-sheet" id="print-sheet">
        <!-- KOP SURAT FORMAL -->
        <header class="kop-header">
          <div class="kop-emblem-box">
            <span class="kop-icon">⚙️</span>
          </div>
          <div class="kop-info">
            <h1 class="kop-title">MAJELIS SHOLAWAT & UNIT SOSIAL WELDING 2</h1>
            <p class="kop-subtitle">Divisi Fabrikasi Industri & Workshop Welding 2 • Pengurus Kas & Kesejahteraan Anggota</p>
            <p class="kop-detail">Bekasi - Jawa Barat • Dokumen Resmi Pembukuan & Laporan Pertanggungjawaban</p>
          </div>
        </header>

        <div class="kop-line-double"></div>

        <!-- JUDUL LAPORAN -->
        <div class="doc-title-block">
          <h2 class="doc-main-title">LAPORAN REKAPITULASI ARUS KAS KEUANGAN</h2>
          <p class="doc-period">PERIODE: {labelPeriode}</p>
          
          <div class="doc-metadata-bar">
            <div><span>No. Dokumen:</span> <strong>{nomorDokumen}</strong></div>
            <div><span>Status Pembukuan:</span> <strong>LENGKAP & SEIMBANG</strong></div>
            <div><span>Tanggal Terbit:</span> <strong>{tanggalHariIni}</strong></div>
          </div>
        </div>

        <!-- RINGKASAN FINANSIAL EKSEKUTIF -->
        <div class="summary-pills-row">
          <div class="summary-box-sheet box-in">
            <div class="sb-label">TOTAL PENERIMAAN (MASUK)</div>
            <div class="sb-value text-emerald">{formatRp(totalMasuk)}</div>
            <div class="sb-sub">Akumulasi seluruh sumber dana</div>
          </div>
          <div class="summary-box-sheet box-out">
            <div class="sb-label">TOTAL PENGELUARAN (KELUAR)</div>
            <div class="sb-value text-rose">{formatRp(totalKeluar)}</div>
            <div class="sb-sub">Total pengeluaran operasional</div>
          </div>
          <div class="summary-box-sheet box-bal">
            <div class="sb-label">SISA SALDO KAS BERJALAN</div>
            <div class="sb-value text-dark">{formatRp(saldoAkhir)}</div>
            <div class="sb-sub">Posisi saldo kas terkini</div>
          </div>
        </div>

        <!-- TABEL FORMAL REKAPITULASI -->
        <div class="sheet-table-wrap">
          <table class="sheet-table">
            <thead>
              <tr>
                <th style="width: 32px;">NO</th>
                <th style="width: 95px;">TANGGAL</th>
                <th style="width: 105px;">KATEGORI</th>
                <th>KETERANGAN TRANSAKSI</th>
                <th style="width: 95px;" class="text-right">PENERIMAAN</th>
                <th style="width: 95px;" class="text-right">PENGELUARAN</th>
                <th style="width: 105px;" class="text-right">SALDO AKHIR</th>
              </tr>
            </thead>
            <tbody>
              {#if processedData.length === 0}
                <tr>
                  <td colspan="7" class="text-center py-4"><em>Tidak ada transaksi tercatat pada periode ini.</em></td>
                </tr>
              {:else}
                {#each processedData as r}
                  <tr>
                    <td class="text-center font-mono">{r.no}</td>
                    <td class="text-center font-mono">{formatTanggalIndo(r.tanggal)}</td>
                    <td>
                      <span class="sheet-cat-badge {r.tipe}">{r.kategori}</span>
                    </td>
                    <td class="desc-sheet font-medium">{r.keterangan}</td>
                    <td class="text-right font-mono val-in">
                      {r.masuk > 0 ? formatRp(r.masuk) : '-'}
                    </td>
                    <td class="text-right font-mono val-out">
                      {r.keluar > 0 ? formatRp(r.keluar) : '-'}
                    </td>
                    <td class="text-right font-mono font-bold">
                      {formatRp(r.runningBalance)}
                    </td>
                  </tr>
                {/each}
              {/if}
            </tbody>
            <tfoot>
              <tr class="sheet-total-row">
                <td colspan="4" class="text-right font-bold">TOTAL KESELURUHAN PERIODE:</td>
                <td class="text-right font-mono font-bold val-in">{formatRp(totalMasuk)}</td>
                <td class="text-right font-mono font-bold val-out">{formatRp(totalKeluar)}</td>
                <td class="text-right font-mono font-bold">{formatRp(saldoAkhir)}</td>
              </tr>
            </tfoot>
          </table>
        </div>

        <!-- CATATAN DOKUMEN -->
        <div class="sheet-notes">
          <h4>Ketentuan & Validasi Dokumen:</h4>
          <ul>
            <li>Laporan ini disusun dengan pencatatan ganda dan sah sebagai dokumen pertanggungjawaban kas resmi.</li>
            <li>Seluruh pengeluaran dan pemasukan telah melalui verifikasi nota, kwitansi, atau bukti transaksi perbankan.</li>
            <li>Sisa saldo kas tercatat telah disesuaikan dengan fisik kas tunai dan rekening bersama pengurus.</li>
          </ul>
        </div>

        <!-- LEMBAR TANDA TANGAN -->
        <footer class="sheet-signatures">
          <div class="sig-block">
            <p class="sig-title">Mengetahui & Menyetujui,</p>
            <p class="sig-role">Ketua Majelis Sholawat Welding 2</p>
            <div class="sig-space"></div>
            <p class="sig-name">( {namaKetua} )</p>
            <p class="sig-id">ID: WLD-2024-001</p>
          </div>

          <div class="sig-block">
            <p class="sig-title">{kotaCetak}, {tanggalHariIni}</p>
            <p class="sig-role">Bendahara Kas & Pembukuan</p>
            <div class="sig-space"></div>
            <p class="sig-name">( {namaBendahara} )</p>
            <p class="sig-id">ID: WLD-2024-018</p>
          </div>
        </footer>

        <!-- FOOTER LEMBAR -->
        <div class="sheet-footer-line">
          <span>Sistem Laporan Keuangan Divisi Welding 2 • Standar Cetak A4</span>
          <span>Dokumen Resmi • Halaman 1 dari 1</span>
        </div>
      </article>
    </div>
  </section>

  <!-- ==============================================================
       MODAL POPUP TAMBAH TRANSAKSI (INTERAKTIF & CEPAT)
       ============================================================== -->
  {#if showModal}
    <!-- svelte-ignore a11y_click_events_have_key_events a11y_no_noninteractive_element_interactions -->
    <div class="modal-overlay no-print" onclick={() => showModal = false} role="presentation">
      <div class="modal-box" onclick={(e) => e.stopPropagation()} role="dialog" aria-modal="true" tabindex="-1">
        <div class="modal-box-header">
          <div class="modal-box-title">
            <span class="m-icon">✍️</span>
            <div>
              <h3>Catat Transaksi Keuangan</h3>
              <p>Tambahkan penerimaan atau pengeluaran kas baru</p>
            </div>
          </div>
          <button class="m-close-btn" onclick={() => showModal = false}>✕</button>
        </div>

        <form onsubmit={submitTambah} class="modal-box-body">
          <!-- TIPE TABS -->
          <div class="type-selector-tab">
            <button
              type="button"
              class="type-tab-btn {formTipe === 'masuk' ? 'active-in' : ''}"
              onclick={() => { formTipe = 'masuk'; formKategori = 'Infaq Rutin'; }}
            >
              📥 Penerimaan (Masuk)
            </button>
            <button
              type="button"
              class="type-tab-btn {formTipe === 'keluar' ? 'active-out' : ''}"
              onclick={() => { formTipe = 'keluar'; formKategori = 'Konsumsi'; }}
            >
              📤 Pengeluaran (Keluar)
            </button>
          </div>

          <div class="form-row-2">
            <div class="form-field">
              <label for="tx-date">Tanggal Transaksi</label>
              <input id="tx-date" type="date" bind:value={formTanggal} required />
            </div>

            <div class="form-field">
              <label for="tx-cat">Kategori</label>
              <select id="tx-cat" bind:value={formKategori}>
                {#if formTipe === 'masuk'}
                  <option value="Infaq Rutin">🕌 Infaq Rutin Sholawat</option>
                  <option value="Donatur">🤲 Donatur / Hamba Allah</option>
                  <option value="Kas Masuk">💼 Kas Masuk Line Welding</option>
                  <option value="Penerimaan Lain">📥 Penerimaan Lain</option>
                {:else}
                  <option value="Konsumsi">☕ Konsumsi Jamaah & Kopi</option>
                  <option value="Operasional">⚙️ Operasional Alat & Gas</option>
                  <option value="Bisyaroh">👳 Bisyaroh Habaib / Guru</option>
                  <option value="Maintenance">🛠️ Service Mesin Las</option>
                  <option value="Sosial">🤝 Santunan Sosial</option>
                {/if}
              </select>
            </div>
          </div>

          <div class="form-field">
            <label for="tx-desc">Keterangan Detail Transaksi</label>
            <input
              id="tx-desc"
              type="text"
              placeholder="Misal: Infaq Jumat Shift Pagi, Beli sarung tangan..."
              bind:value={formKeterangan}
              required
            />
          </div>

          <div class="form-field">
            <label for="tx-amount">Nominal Rupiah (Rp)</label>
            <input
              id="tx-amount"
              type="number"
              min="1000"
              step="1000"
              placeholder="Contoh: 150000"
              bind:value={formNominal}
              required
            />
            <!-- QUICK CHIP BUTTONS -->
            <div class="nominal-chips-bar">
              <span class="chips-label">Cepat:</span>
              <button type="button" class="chip-add" onclick={() => tambahChipNominal(20000)}>+20rb</button>
              <button type="button" class="chip-add" onclick={() => tambahChipNominal(50000)}>+50rb</button>
              <button type="button" class="chip-add" onclick={() => tambahChipNominal(100000)}>+100rb</button>
              <button type="button" class="chip-add" onclick={() => tambahChipNominal(500000)}>+500rb</button>
              <button type="button" class="chip-reset" onclick={() => formNominal = ''}>↺ Hapus</button>
            </div>
          </div>

          <div class="modal-box-footer">
            <button type="button" class="btn-cancel" onclick={() => showModal = false}>
              Batal
            </button>
            <button type="submit" class="btn-submit {formTipe}">
              💾 Simpan Transaksi
            </button>
          </div>
        </form>
      </div>
    </div>
  {/if}
</div>

<style>
  /* ==============================================================
     DESIGN SYSTEM & MODERN INTERACTIVE STYLES
     ============================================================== */
  .interactive-app {
    min-height: 100vh;
    background: #f8fafc;
    color: #0f172a;
    font-family: 'Plus Jakarta Sans', -apple-system, BlinkMacSystemFont, sans-serif;
  }

  /* FLOATING TOAST NOTIFICATION */
  .toast-bubble {
    position: fixed;
    top: 24px;
    right: 24px;
    z-index: 99999;
    background: #0f172a;
    color: #ffffff;
    padding: 12px 20px;
    border-radius: 12px;
    box-shadow: 0 12px 30px rgba(0, 0, 0, 0.2);
    display: flex;
    align-items: center;
    gap: 10px;
    font-weight: 700;
    font-size: 13px;
    animation: slideDown 0.25s cubic-bezier(0.16, 1, 0.3, 1);
  }
  .toast-bubble.error {
    background: #be123c;
  }
  @keyframes slideDown {
    from { opacity: 0; transform: translateY(-12px); }
    to { opacity: 1; transform: translateY(0); }
  }

  /* TOP NAVBAR */
  .top-navbar {
    background: #ffffff;
    border-bottom: 1px solid #e2e8f0;
    position: sticky;
    top: 0;
    z-index: 1000;
    box-shadow: 0 2px 10px rgba(15, 23, 42, 0.04);
  }

  .nav-container {
    max-width: 1280px;
    margin: 0 auto;
    padding: 12px 24px;
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 16px;
    flex-wrap: wrap;
  }

  .brand-group {
    display: flex;
    align-items: center;
    gap: 12px;
  }

  .brand-logo {
    width: 42px;
    height: 42px;
    background: linear-gradient(135deg, #059669, #064e3b);
    border-radius: 12px;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 20px;
    box-shadow: 0 4px 12px rgba(5, 150, 105, 0.25);
  }

  .brand-title {
    font-weight: 800;
    font-size: 15px;
    color: #0f172a;
    letter-spacing: 0.5px;
  }

  .brand-tag {
    font-size: 11px;
    color: #64748b;
    font-weight: 600;
  }

  /* TABS PILL SWITCHER */
  .tab-pill-group {
    display: flex;
    background: #f1f5f9;
    padding: 4px;
    border-radius: 12px;
    gap: 4px;
    border: 1px solid #e2e8f0;
  }

  .tab-pill {
    padding: 8px 18px;
    border: none;
    border-radius: 9px;
    font-size: 12.5px;
    font-weight: 700;
    cursor: pointer;
    background: transparent;
    color: #64748b;
    transition: all 0.2s ease;
  }

  .tab-pill.active {
    background: #ffffff;
    color: #059669;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.06);
  }

  .tab-pill:hover:not(.active) {
    color: #0f172a;
  }

  /* TOP ACTION BUTTONS */
  .nav-actions {
    display: flex;
    align-items: center;
    gap: 10px;
  }

  .action-btn {
    padding: 9px 18px;
    border-radius: 10px;
    font-size: 13px;
    font-weight: 700;
    border: none;
    cursor: pointer;
    transition: all 0.2s ease;
    display: inline-flex;
    align-items: center;
    gap: 6px;
  }

  .btn-print-quick {
    background: #0f172a;
    color: #ffffff;
  }
  .btn-print-quick:hover {
    background: #1e293b;
    transform: translateY(-1px);
  }

  .btn-add-quick {
    background: #059669;
    color: #ffffff;
    box-shadow: 0 4px 12px rgba(5, 150, 105, 0.25);
  }
  .btn-add-quick:hover {
    background: #047857;
    transform: translateY(-1px);
  }

  /* ==============================================================
     DASHBOARD SCREEN (MODERN FINTECH STYLE)
     ============================================================== */
  .dashboard-screen {
    padding: 24px 20px 60px 20px;
  }

  .screen-container {
    max-width: 1280px;
    margin: 0 auto;
    display: flex;
    flex-direction: column;
    gap: 22px;
  }

  /* HERO CARDS GRID */
  .hero-balance-section {
    display: grid;
    grid-template-columns: 2fr 1.1fr 1.1fr;
    gap: 16px;
  }

  .hero-card {
    background: #ffffff;
    border-radius: 20px;
    padding: 22px;
    border: 1px solid #e2e8f0;
    box-shadow: 0 4px 15px rgba(15, 23, 42, 0.03);
    position: relative;
    overflow: hidden;
    transition: transform 0.2s, box-shadow 0.2s;
  }
  .hero-card:hover {
    box-shadow: 0 8px 25px rgba(15, 23, 42, 0.06);
  }

  /* MAIN WEALTH CARD (DARK EMERALD GRADIENT) */
  .hero-balance-card {
    background: linear-gradient(135deg, #064e3b 0%, #022c22 100%);
    color: #ffffff;
    display: flex;
    flex-direction: column;
    justify-content: space-between;
    border-color: #065f46;
  }

  .card-glass-glow {
    position: absolute;
    top: -40px;
    right: -40px;
    width: 180px;
    height: 180px;
    background: radial-gradient(circle, rgba(16, 185, 129, 0.3) 0%, transparent 70%);
    pointer-events: none;
  }

  .hero-card-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 10px;
  }

  .pill-badge-gold {
    background: rgba(245, 158, 11, 0.2);
    color: #fde68a;
    font-size: 11px;
    font-weight: 700;
    padding: 4px 12px;
    border-radius: 20px;
    border: 1px solid rgba(245, 158, 11, 0.3);
  }

  .live-date-pill {
    font-size: 11px;
    color: #a7f3d0;
    font-weight: 600;
  }

  .balance-number {
    font-size: 34px;
    font-weight: 800;
    color: #fef08a;
    margin: 18px 0;
    letter-spacing: -0.5px;
    text-shadow: 0 2px 10px rgba(0, 0, 0, 0.3);
  }

  .hero-card-footer {
    display: flex;
    justify-content: space-between;
    align-items: center;
    flex-wrap: wrap;
    gap: 12px;
    border-top: 1px solid rgba(255, 255, 255, 0.1);
    padding-top: 14px;
  }

  .ratio-badge {
    display: flex;
    align-items: center;
    gap: 6px;
    font-size: 11.5px;
    color: #d1fae5;
  }

  .ratio-dot {
    width: 8px;
    height: 8px;
    border-radius: 50%;
    background: #10b981;
    box-shadow: 0 0 8px #10b981;
  }

  .action-mini-group {
    display: flex;
    gap: 8px;
  }

  .mini-btn {
    background: rgba(255, 255, 255, 0.15);
    border: 1px solid rgba(255, 255, 255, 0.2);
    color: #ffffff;
    font-size: 11px;
    font-weight: 700;
    padding: 5px 12px;
    border-radius: 8px;
    cursor: pointer;
    transition: background 0.15s;
  }
  .mini-btn:hover {
    background: rgba(255, 255, 255, 0.25);
  }

  /* STAT MINI CARDS */
  .stat-top {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 12px;
  }

  .stat-icon {
    width: 38px;
    height: 38px;
    border-radius: 10px;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 18px;
  }
  .income-icon { background: #dcfce7; }
  .expense-icon { background: #fee2e2; }

  .stat-trend {
    font-size: 11px;
    font-weight: 700;
    padding: 3px 8px;
    border-radius: 6px;
  }
  .stat-trend.positive { background: #dcfce7; color: #15803d; }
  .stat-trend.negative { background: #fee2e2; color: #b91c1c; }

  .stat-title {
    font-size: 12px;
    font-weight: 700;
    color: #64748b;
    margin-bottom: 4px;
  }

  .stat-value {
    font-size: 20px;
    font-weight: 800;
    margin-bottom: 12px;
  }
  .text-emerald { color: #059669; }
  .text-rose { color: #e11d48; }

  .stat-bar-container {
    width: 100%;
    height: 6px;
    background: #f1f5f9;
    border-radius: 6px;
    overflow: hidden;
    margin-bottom: 6px;
  }

  .stat-bar-fill {
    height: 100%;
    border-radius: 6px;
    transition: width 0.4s ease;
  }
  .income-bar { background: #10b981; }
  .expense-bar { background: #f43f5e; }

  .stat-sub {
    font-size: 10.5px;
    color: #94a3b8;
    font-weight: 600;
  }

  /* INTERACTIVE FILTER BAR */
  .interactive-filter-bar {
    background: #ffffff;
    border: 1px solid #e2e8f0;
    border-radius: 16px;
    padding: 14px 18px;
    display: flex;
    align-items: center;
    gap: 14px;
    flex-wrap: wrap;
    box-shadow: 0 2px 6px rgba(15, 23, 42, 0.02);
  }

  .search-box {
    position: relative;
    flex: 1;
    min-width: 240px;
  }

  .search-glyph {
    position: absolute;
    left: 12px;
    top: 50%;
    transform: translateY(-50%);
    font-size: 13px;
    color: #94a3b8;
  }

  .search-box input {
    width: 100%;
    padding: 10px 32px 10px 34px;
    background: #f8fafc;
    border: 1px solid #cbd5e1;
    border-radius: 10px;
    font-size: 13px;
    color: #0f172a;
    outline: none;
    transition: border 0.15s;
  }
  .search-box input:focus {
    border-color: #059669;
    background: #ffffff;
  }

  .clear-search {
    position: absolute;
    right: 10px;
    top: 50%;
    transform: translateY(-50%);
    background: none;
    border: none;
    color: #94a3b8;
    cursor: pointer;
    font-size: 12px;
  }

  .chip-filter-row {
    display: flex;
    gap: 6px;
  }

  .filter-chip {
    padding: 8px 14px;
    border-radius: 8px;
    font-size: 12px;
    font-weight: 700;
    cursor: pointer;
    border: 1px solid #e2e8f0;
    background: #ffffff;
    color: #64748b;
    transition: all 0.15s ease;
  }
  .filter-chip.active {
    background: #0f172a;
    color: #ffffff;
    border-color: #0f172a;
  }
  .filter-chip.chip-in.active {
    background: #059669;
    border-color: #059669;
  }
  .filter-chip.chip-out.active {
    background: #e11d48;
    border-color: #e11d48;
  }

  .styled-select {
    padding: 9px 14px;
    border: 1px solid #cbd5e1;
    border-radius: 10px;
    background: #ffffff;
    font-size: 12.5px;
    font-weight: 600;
    color: #334155;
    outline: none;
  }

  .quick-view-switch {
    margin-left: auto;
  }

  .btn-preview-switch {
    background: #f1f5f9;
    color: #0f172a;
    border: 1px solid #cbd5e1;
    padding: 8px 16px;
    border-radius: 9px;
    font-size: 12px;
    font-weight: 700;
    cursor: pointer;
    transition: all 0.15s;
  }
  .btn-preview-switch:hover {
    background: #059669;
    color: #ffffff;
    border-color: #059669;
  }

  /* TRANSACTIONS VIEW TABLE */
  .transactions-view {
    background: #ffffff;
    border: 1px solid #e2e8f0;
    border-radius: 20px;
    padding: 22px;
    box-shadow: 0 4px 16px rgba(15, 23, 42, 0.03);
  }

  .view-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 16px;
  }

  .view-title {
    font-size: 16px;
    font-weight: 800;
    color: #0f172a;
    margin: 0;
  }

  .view-subtitle {
    font-size: 12px;
    color: #64748b;
    margin: 2px 0 0 0;
  }

  .tool-btn {
    background: none;
    border: 1px solid #e2e8f0;
    padding: 6px 12px;
    border-radius: 8px;
    font-size: 11.5px;
    color: #64748b;
    font-weight: 600;
    cursor: pointer;
  }
  .tool-btn:hover {
    background: #f8fafc;
    color: #0f172a;
  }

  .table-card-wrapper {
    overflow-x: auto;
  }

  .interactive-table {
    width: 100%;
    border-collapse: separate;
    border-spacing: 0;
    font-size: 13px;
  }

  .interactive-table th {
    background: #f8fafc;
    color: #475569;
    font-weight: 700;
    font-size: 11px;
    letter-spacing: 0.5px;
    text-transform: uppercase;
    padding: 12px 14px;
    border-bottom: 2px solid #e2e8f0;
    text-align: left;
  }

  .interactive-table td {
    padding: 14px 14px;
    border-bottom: 1px solid #f1f5f9;
    vertical-align: middle;
  }

  .tx-row:hover td {
    background: #f8fafc;
  }

  .cat-pill {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 4px 10px;
    border-radius: 20px;
    font-size: 11px;
    font-weight: 700;
  }
  .cat-pill.masuk { background: #dcfce7; color: #15803d; }
  .cat-pill.keluar { background: #fee2e2; color: #b91c1c; }

  .val-in { color: #15803d; }
  .val-out { color: #e11d48; }
  .val-running { color: #0f172a; }

  .action-del-btn {
    background: none;
    border: none;
    cursor: pointer;
    font-size: 13px;
    opacity: 0.4;
    transition: opacity 0.15s;
  }
  .action-del-btn:hover {
    opacity: 1;
  }

  .empty-state {
    text-align: center;
    padding: 40px 10px;
  }
  .empty-emoji {
    font-size: 36px;
    margin-bottom: 8px;
  }
  .empty-state p {
    color: #64748b;
    font-size: 13px;
    margin-bottom: 14px;
  }
  .btn-clear-filter {
    background: #0f172a;
    color: #ffffff;
    border: none;
    padding: 8px 16px;
    border-radius: 8px;
    font-size: 12px;
    font-weight: 700;
    cursor: pointer;
  }

  /* ==============================================================
     VIEW 2: PRATINJAU DOKUMEN CETAK A4 (SCREEN VIEW)
     ============================================================== */
  .print-document-screen {
    padding-bottom: 60px;
  }

  .hidden-on-screen {
    display: none;
  }

  .preview-toolbar {
    background: #0f172a;
    color: #ffffff;
    padding: 14px 24px;
    margin-bottom: 24px;
    box-shadow: 0 4px 15px rgba(0, 0, 0, 0.15);
  }

  .toolbar-content {
    max-width: 1200px;
    margin: 0 auto;
    display: flex;
    justify-content: space-between;
    align-items: center;
    flex-wrap: wrap;
    gap: 12px;
  }

  .paper-badge {
    background: #059669;
    padding: 3px 10px;
    border-radius: 6px;
    font-size: 11px;
    font-weight: 800;
    margin-right: 8px;
  }

  .toolbar-btns {
    display: flex;
    gap: 10px;
  }

  .btn-action-primary {
    background: #10b981;
    color: #064e3b;
    border: none;
    padding: 9px 18px;
    border-radius: 8px;
    font-size: 12.5px;
    font-weight: 800;
    cursor: pointer;
    box-shadow: 0 2px 8px rgba(16, 185, 129, 0.3);
  }
  .btn-action-primary:hover {
    background: #34d399;
  }

  .btn-action-secondary {
    background: rgba(255, 255, 255, 0.15);
    color: #ffffff;
    border: 1px solid rgba(255, 255, 255, 0.2);
    padding: 9px 16px;
    border-radius: 8px;
    font-size: 12.5px;
    font-weight: 700;
    cursor: pointer;
  }

  /* PAPER SHEET CONTAINER */
  .a4-sheet-wrapper {
    display: flex;
    justify-content: center;
    padding: 10px 16px;
  }

  .a4-paper-sheet {
    background: #ffffff;
    width: 210mm;
    min-height: 297mm;
    padding: 16mm 18mm 16mm 18mm;
    box-shadow: 0 10px 35px rgba(15, 23, 42, 0.12);
    border: 1px solid #cbd5e1;
    box-sizing: border-box;
    display: flex;
    flex-direction: column;
    font-size: 8.5pt;
    line-height: 1.4;
  }

  /* KOP SURAT FORMAL */
  .kop-header {
    display: flex;
    align-items: center;
    gap: 16px;
    padding-bottom: 10px;
  }

  .kop-emblem-box {
    width: 60px;
    height: 60px;
    border: 2px solid #059669;
    border-radius: 50%;
    display: flex;
    align-items: center;
    justify-content: center;
    background: #ecfdf5;
  }
  .kop-icon { font-size: 32px; }

  .kop-info {
    flex: 1;
    text-align: center;
  }

  .kop-title {
    font-size: 15pt;
    font-weight: 800;
    color: #064e3b;
    margin: 0;
    letter-spacing: 0.5px;
  }
  .kop-subtitle {
    font-size: 8pt;
    color: #334155;
    margin: 3px 0 1px 0;
    font-weight: 600;
  }
  .kop-detail {
    font-size: 7.5pt;
    color: #64748b;
    margin: 0;
  }

  .kop-line-double {
    border-bottom: 3px solid #0f172a;
    border-top: 1px solid #0f172a;
    height: 3px;
    margin-bottom: 14px;
  }

  /* DOC TITLE BLOCK */
  .doc-title-block {
    text-align: center;
    margin-bottom: 14px;
  }
  .doc-main-title {
    font-size: 13pt;
    font-weight: 800;
    color: #0f172a;
    letter-spacing: 0.8px;
    margin: 0 0 3px 0;
  }
  .doc-period {
    font-size: 9.5pt;
    font-weight: 700;
    color: #059669;
    margin: 0 0 8px 0;
  }

  .doc-metadata-bar {
    display: flex;
    justify-content: space-between;
    background: #f8fafc;
    border: 1px solid #e2e8f0;
    padding: 6px 12px;
    border-radius: 6px;
    font-size: 8pt;
    color: #475569;
  }

  /* SUMMARY PILLS IN SHEET */
  .summary-pills-row {
    display: grid;
    grid-template-columns: 1fr 1fr 1fr;
    gap: 10px;
    margin-bottom: 14px;
  }

  .summary-box-sheet {
    border: 1px solid #cbd5e1;
    border-radius: 6px;
    padding: 8px 12px;
    background: #ffffff;
  }
  .box-in { background: #f0fdf4; border-color: #86efac; }
  .box-out { background: #fef2f2; border-color: #fca5a5; }
  .box-bal { background: #f8fafc; border-color: #94a3b8; }

  .sb-label {
    font-size: 7pt;
    font-weight: 700;
    color: #475569;
    letter-spacing: 0.5px;
  }
  .sb-value {
    font-size: 12.5pt;
    font-weight: 800;
    font-family: ui-monospace, monospace;
    line-height: 1.2;
  }
  .sb-sub {
    font-size: 6.5pt;
    color: #64748b;
  }

  /* SHEET TABLE */
  .sheet-table-wrap {
    flex: 1;
    margin-bottom: 12px;
  }

  .sheet-table {
    width: 100%;
    border-collapse: collapse;
    font-size: 7.5pt;
  }

  .sheet-table th,
  .sheet-table td {
    border: 1px solid #cbd5e1;
    padding: 4.5px 6px;
  }

  .sheet-table thead th {
    background: #0f172a;
    color: #ffffff;
    font-weight: 700;
    text-align: center;
  }

  .sheet-table tbody tr:nth-child(even) {
    background: #f8fafc;
  }

  .sheet-cat-badge {
    display: inline-block;
    padding: 1px 5px;
    border-radius: 4px;
    font-size: 6.5pt;
    font-weight: 700;
  }
  .sheet-cat-badge.masuk { background: #dcfce7; color: #166534; }
  .sheet-cat-badge.keluar { background: #fee2e2; color: #991b1b; }

  .sheet-total-row {
    background: #f1f5f9;
    border-top: 2px solid #0f172a;
  }

  /* SHEET NOTES */
  .sheet-notes {
    border: 1px dashed #cbd5e1;
    border-radius: 4px;
    padding: 7px 12px;
    background: #fafafa;
    margin-bottom: 16px;
    font-size: 7pt;
    color: #475569;
  }
  .sheet-notes h4 {
    margin: 0 0 3px 0;
    font-size: 7.5pt;
    font-weight: 700;
    color: #0f172a;
  }
  .sheet-notes ul {
    margin: 0;
    padding-left: 14px;
  }

  /* SIGNATURES */
  .sheet-signatures {
    display: flex;
    justify-content: space-between;
    padding: 0 24px;
    margin-top: 6px;
  }

  .sig-block {
    text-align: center;
    width: 200px;
  }
  .sig-title { font-size: 7.5pt; color: #64748b; margin: 0; }
  .sig-role { font-size: 8pt; font-weight: 700; color: #0f172a; margin: 2px 0 0 0; }
  .sig-space { height: 48px; }
  .sig-name { font-size: 8.5pt; font-weight: 800; text-decoration: underline; color: #0f172a; margin: 0; }
  .sig-id { font-size: 7pt; color: #64748b; margin: 2px 0 0 0; }

  .sheet-footer-line {
    margin-top: 14px;
    padding-top: 4px;
    border-top: 1px solid #e2e8f0;
    display: flex;
    justify-content: space-between;
    font-size: 6.5pt;
    color: #94a3b8;
  }

  /* ==============================================================
     MODAL POPUP TAMBAH TRANSAKSI
     ============================================================== */
  .modal-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(15, 23, 42, 0.6);
    backdrop-filter: blur(5px);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 9999;
    padding: 16px;
  }

  .modal-box {
    background: #ffffff;
    width: 100%;
    max-width: 480px;
    border-radius: 20px;
    box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.25);
    overflow: hidden;
    animation: scaleUp 0.2s cubic-bezier(0.16, 1, 0.3, 1);
  }
  @keyframes scaleUp {
    from { opacity: 0; transform: scale(0.95); }
    to { opacity: 1; transform: scale(1); }
  }

  .modal-box-header {
    background: #f8fafc;
    border-bottom: 1px solid #e2e8f0;
    padding: 16px 22px;
    display: flex;
    justify-content: space-between;
    align-items: center;
  }

  .modal-box-title {
    display: flex;
    align-items: center;
    gap: 12px;
  }
  .m-icon {
    font-size: 26px;
  }
  .modal-box-title h3 {
    margin: 0;
    font-size: 15px;
    font-weight: 800;
  }
  .modal-box-title p {
    margin: 0;
    font-size: 11.5px;
    color: #64748b;
  }

  .m-close-btn {
    background: none;
    border: none;
    font-size: 18px;
    color: #94a3b8;
    cursor: pointer;
  }

  .modal-box-body {
    padding: 20px 22px;
    display: flex;
    flex-direction: column;
    gap: 14px;
  }

  /* TABS MASUK/KELUAR SELECTOR */
  .type-selector-tab {
    display: grid;
    grid-template-columns: 1fr 1fr;
    background: #f1f5f9;
    padding: 4px;
    border-radius: 12px;
    gap: 6px;
  }

  .type-tab-btn {
    padding: 9px;
    border: none;
    border-radius: 8px;
    font-size: 12.5px;
    font-weight: 700;
    cursor: pointer;
    background: transparent;
    color: #64748b;
    transition: all 0.15s;
  }
  .type-tab-btn.active-in {
    background: #059669;
    color: #ffffff;
    box-shadow: 0 2px 8px rgba(5, 150, 105, 0.25);
  }
  .type-tab-btn.active-out {
    background: #e11d48;
    color: #ffffff;
    box-shadow: 0 2px 8px rgba(225, 29, 72, 0.25);
  }

  .form-row-2 {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 12px;
  }

  .form-field {
    display: flex;
    flex-direction: column;
    gap: 4px;
  }
  .form-field label {
    font-size: 11.5px;
    font-weight: 700;
    color: #334155;
  }
  .form-field input,
  .form-field select {
    padding: 9px 12px;
    border: 1px solid #cbd5e1;
    border-radius: 8px;
    font-size: 13px;
    outline: none;
  }
  .form-field input:focus,
  .form-field select:focus {
    border-color: #059669;
  }

  .nominal-chips-bar {
    display: flex;
    align-items: center;
    gap: 6px;
    flex-wrap: wrap;
    margin-top: 6px;
  }
  .chips-label {
    font-size: 11px;
    font-weight: 700;
    color: #64748b;
  }
  .chip-add {
    background: #f1f5f9;
    border: 1px solid #cbd5e1;
    border-radius: 6px;
    font-size: 11px;
    font-weight: 700;
    color: #0f172a;
    padding: 4px 8px;
    cursor: pointer;
  }
  .chip-add:hover {
    background: #e2e8f0;
  }
  .chip-reset {
    background: none;
    border: none;
    font-size: 11px;
    font-weight: 700;
    color: #e11d48;
    cursor: pointer;
  }

  .modal-box-footer {
    display: flex;
    justify-content: flex-end;
    gap: 10px;
    margin-top: 8px;
  }
  .btn-cancel {
    background: #f1f5f9;
    border: 1px solid #e2e8f0;
    color: #64748b;
    padding: 9px 16px;
    border-radius: 8px;
    font-size: 12.5px;
    font-weight: 700;
    cursor: pointer;
  }
  .btn-submit {
    color: #ffffff;
    border: none;
    padding: 9px 20px;
    border-radius: 8px;
    font-size: 12.5px;
    font-weight: 800;
    cursor: pointer;
  }
  .btn-submit.masuk { background: #059669; }
  .btn-submit.keluar { background: #e11d48; }

  /* ==============================================================
     HELPER CLASSES
     ============================================================== */
  .text-center { text-align: center; }
  .text-right { text-align: right; }
  .font-mono { font-family: ui-monospace, SFMono-Regular, Menlo, Monaco, Consolas, monospace; }
  .font-bold { font-weight: 700; }
  .font-medium { font-weight: 500; }
  .text-date { font-size: 12px; color: #475569; }
  .num-sub { color: #94a3b8; font-weight: 600; }

  /* ==============================================================
     MEDIA CETAK (A4 STRICT PRINT RULES)
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

    .interactive-app {
      background: #ffffff !important;
      min-height: auto !important;
    }

    .no-print {
      display: none !important;
    }

    .hidden-on-screen {
      display: block !important;
    }

    .print-document-screen {
      padding: 0 !important;
      margin: 0 !important;
      display: block !important;
    }

    .a4-sheet-wrapper {
      padding: 0 !important;
      margin: 0 !important;
      display: block !important;
    }

    .a4-paper-sheet {
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

    .sheet-signatures {
      page-break-inside: avoid !important;
      break-inside: avoid !important;
    }
  }

  /* RESPONSIVE DESIGN (MOBILE / TABLET) */
  @media screen and (max-width: 960px) {
    .hero-balance-section {
      grid-template-columns: 1fr;
    }
    .top-navbar .nav-container {
      flex-direction: column;
      align-items: stretch;
      gap: 12px;
    }
    .brand-group {
      justify-content: center;
    }
    .tab-pill-group {
      justify-content: center;
    }
    .nav-actions {
      justify-content: center;
    }
    .interactive-filter-bar {
      flex-direction: column;
      align-items: stretch;
    }
    .quick-view-switch {
      margin-left: 0;
    }
    .btn-preview-switch {
      width: 100%;
      text-align: center;
    }
    .a4-paper-sheet {
      width: 100% !important;
      min-height: auto !important;
      padding: 14px 10px !important;
    }
    .summary-pills-row {
      grid-template-columns: 1fr;
    }
    .sheet-signatures {
      flex-direction: column;
      align-items: center;
      gap: 20px;
    }
  }
</style>
