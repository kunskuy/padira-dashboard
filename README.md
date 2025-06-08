# Padira Dashboard

Dashboard admin untuk memantau data petani, penggilingan, dan distributor dalam ekosistem pertanian.

## Prerequisites

- Node.js (versi 16 atau lebih baru)
- npm atau yarn

## Project Setup

### 1. Install dependencies
```bash
npm install
```

### 2. Environment Configuration
Salin file `.env.example` ke `.env` dan isi dengan konfigurasi Firebase Anda:

```bash
cp .env.example .env
```

Edit file `.env` dengan konfigurasi Firebase Anda:
```env
VITE_FIREBASE_API_KEY=your_firebase_api_key_here
VITE_FIREBASE_AUTH_DOMAIN=your_project_id.firebaseapp.com
VITE_FIREBASE_DATABASE_URL=https://your_project_id.firebaseio.com
VITE_FIREBASE_PROJECT_ID=your_project_id
VITE_FIREBASE_STORAGE_BUCKET=your_project_id.appspot.com
VITE_FIREBASE_MESSAGING_SENDER_ID=your_messaging_sender_id
VITE_FIREBASE_APP_ID=your_app_id
VITE_FIREBASE_MEASUREMENT_ID=your_measurement_id
```

## Development

### Compiles and hot-reloads for development
```bash
npm run dev
```

### Preview production build
```bash
npm run preview
```

## Production

### Compiles and minifies for production
```bash
npm run build
```

## Code Quality

### Lints and fixes files
```bash
npm run lint
```

## Tech Stack

- **Frontend Framework**: Vue.js 3
- **Build Tool**: Vite
- **Backend**: Firebase
- **Charts**: Chart.js
- **PDF Generation**: jsPDF + jsPDF-AutoTable
- **Icons**: Lucide Vue Next
- **File Processing**: XLSX
- **Code Quality**: ESLint

## Project Structure

```
padira-dashboard/
├── public/
├── src/
│   ├── components/
│   ├── views/
│   ├── assets/
│   └── main.js
├── .env.example
├── package.json
├── vite.config.js
└── README.md
```

## Configuration

Untuk konfigurasi lebih lanjut, lihat [Vite Configuration Reference](https://vitejs.dev/config/).