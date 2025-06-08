<template>
  <main class="container" role="main" aria-label="Data Hasil Penggilingan">
    <div class="headline-card">
      <h1 class="headline">Data Hasil Penggilingan</h1>
    </div>

    <!-- Table Section -->
    <section class="table-section" aria-labelledby="table-title">
      <header class="table-header">
        <h2 id="table-title" class="section-title">Daftar Hasil Penggilingan</h2>
        <div class="button-group">
          <button @click="downloadExcel" class="download-button" :disabled="loadingData || penggilinganData.length === 0" 
                  aria-label="Download data sebagai Excel">
            {{ loadingData ? 'Memuat...' : 'Download Excel' }}
          </button>
          <button @click="downloadPDF" class="download-button" :disabled="loadingData || penggilinganData.length === 0" 
                  aria-label="Download data sebagai PDF">
            {{ loadingData ? 'Memuat...' : 'Download PDF' }}
          </button>
          <button @click="refreshData" class="refresh-button" :disabled="loadingData" aria-live="polite"
            aria-disabled="loadingData.toString()">
            {{ loadingData ? 'Memuat...' : 'Refresh' }}
          </button>
        </div>
      </header>

      <div v-if="loadingData" class="loading" aria-live="polite" aria-busy="true">
        Memuat data...
      </div>

      <div v-else-if="penggilinganData.length === 0" class="no-data" role="status">
        Tidak ada data penggilingan yang tersedia
      </div>

      <div v-else class="table-container">
        <table class="price-table" role="table" aria-describedby="table-title">
          <thead>
            <tr>
              <th v-for="(header, index) in headers" :key="index" scope="col">{{ header }}</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(row, rowIndex) in penggilinganData" :key="rowIndex">
              <td v-for="(header, colIndex) in headers" :key="colIndex">
                <span v-if="header === 'Status'"
                      class="status"
                      :class="`status-${row.statusType}`">
                  {{ row.status }}
                </span>
                <template v-else>
                  {{ getValueByHeader(row, header) }}
                </template>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </section>
  </main>
</template>

<script>
import { ref, onValue } from 'firebase/database'
import { database } from '../firebase.js' // Adjust path as needed
import * as XLSX from 'xlsx'
import jsPDF from 'jspdf'
import autoTable from 'jspdf-autotable'

export default {
  name: 'PenggilinganView',
  data() {
    return {
      penggilinganData: [],
      headers: ['ID', 'Tanggal', 'Jenis Beras', 'Deskripsi', 'Hasil Beras (Ton)'],
      loadingData: false
    }
  },
  mounted() {
    this.loadData()
  },
  methods: {
    loadData() {
      this.loadingData = true
      const notesRef = ref(database, 'notes');
      onValue(notesRef, (snapshot) => {
        const data = snapshot.val();
        const filteredData = [];
        
        for (let id in data) {
          if (data[id].userRole === 'Penggilingan') {
            const volume = parseFloat(data[id].weight) / 1000 || 0; // Convert weight to tons
            
            filteredData.push({
              id: id,
              tanggal: data[id].dateString || '',
              jenisBeras: data[id].productType || '',
              kualitas: data[id].description || '',
              hasil: volume.toFixed(2),
            });
          }
        }
        
        this.penggilinganData = filteredData;
        this.loadingData = false
      });
    },
    
    refreshData() {
      this.loadData()
    },
    
    downloadExcel() {
      if (this.penggilinganData.length === 0) {
        alert('Tidak ada data untuk didownload')
        return
      }

      // Siapkan data untuk Excel
      const excelData = this.penggilinganData.map(row => ({
        'ID': row.id,
        'Tanggal': row.tanggal,
        'Jenis Beras': row.jenisBeras,
        'Deskripsi': row.kualitas,
        'Hasil Beras (Ton)': row.hasil
      }))

      // Buat workbook dan worksheet
      const workbook = XLSX.utils.book_new()
      const worksheet = XLSX.utils.json_to_sheet(excelData)

      // Set lebar kolom
      const columnWidths = [
        { wch: 15 }, // ID
        { wch: 12 }, // Tanggal
        { wch: 15 }, // Jenis Beras
        { wch: 20 }, // Deskripsi
        { wch: 18 }  // Hasil Beras (Ton)
      ]
      worksheet['!cols'] = columnWidths

      // Tambahkan worksheet ke workbook
      XLSX.utils.book_append_sheet(workbook, worksheet, 'Data Penggilingan')

      // Generate nama file dengan timestamp
      const now = new Date()
      const timestamp = now.toISOString().slice(0, 19).replace(/[-:]/g, '').replace('T', '_')
      const filename = `Data_Hasil_Penggilingan_${timestamp}.xlsx`

      // Download file
      XLSX.writeFile(workbook, filename)
    },

    downloadPDF() {
      if (this.penggilinganData.length === 0) {
        alert('Tidak ada data untuk didownload')
        return
      }

      // Buat instance jsPDF
      const doc = new jsPDF()

      // Set font untuk bahasa Indonesia
      doc.setFont('helvetica')

      // Judul dokumen
      doc.setFontSize(16)
      doc.setFont('helvetica', 'bold')
      doc.text('DATA HASIL PENGGILINGAN', doc.internal.pageSize.width / 2, 20, { align: 'center' })

      // Tanggal generate
      const now = new Date()
      const dateString = now.toLocaleDateString('id-ID', {
        year: 'numeric',
        month: 'long',
        day: 'numeric'
      })
      doc.setFontSize(10)
      doc.setFont('helvetica', 'normal')
      doc.text(`Digenerate pada: ${dateString}`, doc.internal.pageSize.width / 2, 30, { align: 'center' })

      // Siapkan data untuk tabel
      const tableData = this.penggilinganData.map(row => [
        row.id,
        row.tanggal,
        row.jenisBeras,
        row.kualitas,
        row.hasil
      ])

      // Konfigurasi tabel
      const tableConfig = {
        head: [this.headers],
        body: tableData,
        startY: 40,
        theme: 'striped',
        headStyles: {
          fillColor: [17, 24, 39], // Dark gray
          textColor: [255, 255, 255], // White text
          fontStyle: 'bold',
          fontSize: 10
        },
        bodyStyles: {
          fontSize: 9,
          textColor: [55, 65, 81]
        },
        alternateRowStyles: {
          fillColor: [249, 250, 251] // Light gray
        },
        columnStyles: {
          0: { cellWidth: 25 }, // ID
          1: { cellWidth: 30 }, // Tanggal
          2: { cellWidth: 35 }, // Jenis Beras
          3: { cellWidth: 50 }, // Deskripsi
          4: { cellWidth: 35 }  // Hasil Beras (Ton)
        },
        margin: { top: 40, left: 15, right: 15 },
        styles: {
          overflow: 'linebreak',
          cellPadding: 3,
          fontSize: 9,
          lineColor: [229, 231, 235],
          lineWidth: 0.5
        }
      }

      // Generate tabel
      autoTable(doc, tableConfig)

      // Footer dengan total data
      const finalY = doc.lastAutoTable?.finalY || 40
      doc.setFontSize(10)
      doc.setFont('helvetica', 'bold')
      doc.text(`Total Data: ${this.penggilinganData.length} entri`, 15, finalY + 15)

      // Hitung total hasil beras
      const totalHasil = this.penggilinganData.reduce((sum, row) => {
        return sum + parseFloat(row.hasil)
      }, 0)
      doc.text(`Total Hasil Beras: ${totalHasil.toFixed(2)} Ton`, 15, finalY + 25)

      // Generate nama file dengan timestamp
      const timestamp = now.toISOString().slice(0, 19).replace(/[-:]/g, '').replace('T', '_')
      const filename = `Data_Hasil_Penggilingan_${timestamp}.pdf`

      // Save PDF
      doc.save(filename)
    },
    
    getValueByHeader(row, header) {
      const headerMap = {
        'ID': 'id',
        'Tanggal': 'tanggal',
        'Jenis Beras': 'jenisBeras',
        'Deskripsi': 'kualitas',
        'Hasil Beras (Ton)': 'hasil',
      }
     
      const key = headerMap[header] || header.toLowerCase()
      return row[key]
    }
  }
}
</script>

<style scoped>
/* Reset and base */
*,
*::before,
*::after {
  box-sizing: border-box;
}

.container {
  width: 100%;
  margin: 0;
  padding: 2rem 0;
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen,
    Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
  color: #374151;
  background-color: #f5f5f5;
  line-height: 1.5;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

.headline-card {
  background: #ffffff;
  margin: 0 0 1.5rem 0;
  padding: 2rem;
  border-bottom: 1px solid #e5e7eb;
}

.headline {
  font-size: 1.75rem;
  font-weight: 600;
  margin-bottom: 0;
  color: #111827;
  text-align: center;
  padding: 0;
}

.table-section {
  background: #ffffff;
  margin: 0;
  padding: 2rem;
  border-top: 1px solid #e5e7eb;
}

.section-title {
  font-size: 1.25rem;
  font-weight: 600;
  margin-bottom: 1.5rem;
  color: #111827;
}

.table-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 1.5rem;
}

.button-group {
  display: flex;
  gap: 0.5rem;
}

.refresh-button,
.download-button {
  background-color: #111827;
  color: #ffffff;
  padding: 0.5rem 1rem;
  font-size: 0.75rem;
  font-weight: 500;
  border: none;
  border-radius: 0.375rem;
  cursor: pointer;
  transition: background-color 0.15s ease;
  white-space: nowrap;
}

.refresh-button:hover:not(:disabled),
.download-button:hover:not(:disabled) {
  background-color: #374151;
}

.refresh-button:disabled,
.download-button:disabled {
  background-color: #9ca3af;
  cursor: not-allowed;
}

.loading,
.no-data {
  text-align: center;
  padding: 2rem;
  color: #6b7280;
  font-style: italic;
  font-size: 0.875rem;
}

.table-container {
  overflow-x: auto;
  border-radius: 0.375rem;
  border: 1px solid #e5e7eb;
}

.price-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.875rem;
  background: #ffffff;
}

.price-table th,
.price-table td {
  padding: 0.75rem 1rem;
  text-align: left;
  border-bottom: 1px solid #e5e7eb;
}

.price-table th {
  font-weight: 600;
  color: #111827;
  background-color: #f9fafb;
  user-select: none;
}

.price-table tbody tr:hover {
  background-color: #f9fafb;
}

/* Status Styles */
.status {
  padding: 0.25rem 0.5rem;
  border-radius: 20px;
  font-size: 0.75rem;
  font-weight: 600;
  display: inline-block;
}

.status-success {
  background-color: #e6f7ea;
  color: #2a9d58;
}

.status-warning {
  background-color: #fff4e5;
  color: #e69c00;
}

.status-danger {
  background-color: #feeaea;
  color: #e53e3e;
}

@media (max-width: 768px) {
  .container {
    padding: 1rem 0;
  }

  .headline-card {
    padding: 1.5rem 1rem;
  }

  .headline {
    font-size: 1.5rem;
    padding: 0;
  }

  .table-section {
    padding: 1.5rem 1rem;
  }

  .table-header {
    flex-direction: column;
    gap: 1rem;
    align-items: stretch;
  }

  .button-group {
    flex-direction: column;
    gap: 0.5rem;
  }

  .refresh-button,
  .download-button {
    width: 100%;
  }

  .table-container {
    overflow-x: scroll;
  }
}

@media (max-width: 480px) {
  .button-group {
    flex-direction: column;
  }
}
</style>