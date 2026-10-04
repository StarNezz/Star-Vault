# Star Vault

Fondasi aplikasi personal finance Flutter, local-first, tanpa backend. Lima tab dasar disiapkan; desain final dan formulir belum dibuat.

## Menjalankan

Dibuat dan diverifikasi dengan Flutter 3.47.4 / Dart 3.13.3. Buka folder ini di VS Code.

```sh
flutter pub get
dart run build_runner build
dart format .
flutter analyze
flutter test
flutter run
```

Pada Windows dengan SDK di folder yang mengandung spasi, gunakan `./tool/flutter_windows.ps1 test` atau `./tool/flutter_windows.ps1 build apk --debug` bila native-assets hook gagal. Rincian hasil dan keterbatasan build dicatat di `docs/verification.md`.

Target awal Android dan iOS. Build iOS memerlukan macOS/Xcode. Web dan desktop belum dikonfigurasi. Untuk Android: `flutter build apk --debug`. Application ID sementara: `com.starvault.star_vault`; atur ID dan signing sebelum rilis.

## Struktur

```text
lib/
  main.dart                     # ProviderScope / bootstrap
  app.dart                      # MaterialApp.router dan pengaturan tema
  core/
    constants/                  # String umum
    database/                   # Schema, koneksi, migration, seed Drift
    errors/                     # Error validasi domain
    routing/                    # GoRouter / StatefulShellRoute
    security/                   # Kontrak app lock; adapter secure storage/biometrik
    services/                   # Kontrak backup
    theme/                      # Tokens warna, spacing, radius, typography
    utils/                      # Uang integer dan formatting
    widgets/                    # Navigation scaffold, bottom navigation, placeholder
  features/
    accounts/                   # domain/entities, domain/repositories,
    categories/                 # data/repositories, presentation/pages/providers
    transactions/
    savings/
    profile/
    statistics/                 # Agregasi transaksi domain dan presentation
    security/                   # Placeholder dan provider adapter perangkat
  shared/
    models/                     # Enum dan filter transaksi
    providers/                  # Lifecycle koneksi database
    widgets/                    # Komponen presentasional reusable
test/
  ledger_test.dart               # SQLite nyata in-memory/file sementara
  widget_test.dart               # Router, tab, placeholder, dark theme
```

Folder kosong tidak dibuat sekadar mengikuti diagram. Drift sendiri menjadi local datasource dan menghasilkan model persistence; entity domain tetap terpisah. Tidak ada wrapper datasource yang hanya meneruskan panggilan. Repository dapat diganti/di-override melalui Riverpod untuk penambahan sync. Statistik menggunakan repository transaksi yang sama tanpa database atau repository statistik duplikat.

## Dependencies

Runtime: `flutter_riverpod`, `go_router`, `drift`, `drift_flutter`, `flutter_secure_storage`, `local_auth`, `uuid`.
Dev: `drift_dev`, `build_runner`, `flutter_lints`, `flutter_test`.
Versi hasil resolusi dikunci di `pubspec.lock`. Freezed/json_serializable belum diperlukan: entity immutable sederhana dan model Drift sudah memenuhi kebutuhan tahap awal.

## Database dan aturan uang

Schema versi 1: `accounts`, `categories`, `transactions`, `savings`, `scheduled_transactions`, `app_settings`.

- UUID v4 untuk entity, ID 1 untuk singleton settings.
- Uang disimpan sebagai integer satuan terkecil. Ledger awal IDR memiliki scale 2: Rp10.000 = 1.000.000 sen. Tidak ada konversi melalui `double`.
- Satu mata uang per ledger. Mengganti currency bukan sekadar mengganti label; UI perubahan currency belum aktif.
- Batas nominal operasi ±9.007.199.254.740.991. Saldo negatif diizinkan untuk overdraft/koreksi.
- Akun memiliki `isActive`; total saldo hanya menghitung akun aktif. Saldo awal tidak dihitung sebagai pendapatan.
- Transaksi memiliki `destinationAccountId` untuk transfer. Income/expense wajib berkategori dengan jenis sesuai; transfer tanpa kategori dan memerlukan dua akun aktif berbeda.
- Pembuatan transaksi beserta pembaruan saldo berjalan atomik. Penghapusan membalik perubahan saldo atomik, termasuk kedua sisi transfer dan akun yang sudah diarsipkan.
- Tidak ada API edit transaksi langsung yang melewati rekonsiliasi saldo. Implementasikan update atomik sebelum menambahkan UI edit.
- Foreign key aktif setiap koneksi. Check constraints untuk nominal, enum, dan bentuk transfer; index akun/tanggal, tujuan/tanggal, kategori/tanggal, tanggal, serta jadwal aktif/jatuh tempo.
- Seed hanya pada pembuatan database: Makanan, Kafe, Hiburan, Transportasi, Kesehatan, Pulsa, Laundry, DLL; Pemasukan, Bonus. Tidak ada akun atau transaksi demo.
- Tabungan adalah pelacakan target terpisah, bukan akun pemilik dana; perubahan progress tidak memindahkan saldo.
- Statistik menjumlahkan transaksi tersaring, mengecualikan transfer. Filter mendukung interval hari/bulan/tahun, kategori, akun (kedua sisi transfer), dan jenis. Interval menggunakan awal inklusif dan akhir eksklusif, berdasarkan kalender lokal.
- `scheduled_transactions` baru schema; belum ada scheduler atau eksekusi otomatis.

Sebelum mengubah schema yang sudah dirilis, naikkan `schemaVersion`, tambahkan migrasi non-destruktif dan tes migrasi. Jangan menghapus database pengguna sebagai strategi upgrade. Sinkronisasi kelak memerlukan metadata revision/tombstone/conflict policy; UUID dan repository abstraction saja tidak berarti cloud sync sudah tersedia.

## Routing

| Path | Halaman |
| --- | --- |
| `/accounts` | AccountsPage |
| `/accounts/add` | AddAccountPage |
| `/accounts/:id` | AccountDetailPage |
| `/categories` | CategoriesPage |
| `/categories/add` | AddCategoryPage |
| `/categories/:id/edit` | EditCategoryPage |
| `/operations` | OperationsPage |
| `/operations/add` | AddTransactionPage |
| `/operations/:id` | TransactionDetailPage |
| `/statistics` | StatisticsPage |
| `/profile` | ProfilePage |
| `/profile/security` | SecurityPage |
| `/profile/change-pin` | ChangePinPage |
| `/profile/backup` | BackupRestorePage |
| `/savings/add` | AddSavingPage |
| `/savings/:id` | SavingDetailPage |

StatefulShellRoute.indexedStack mempertahankan navigator/tab. List menggunakan PageStorageKey. Halaman detail menerima ID namun belum memuat detail; form dan action save belum ada. Rute tab dan halaman turunannya memiliki bottom navigation; tabungan dibuka di navigator utama.

## Komponen dan provider

Komponen: AppBalanceHeader, MonthSelector, FinanceCard, CategoryCard, AccountCard, SavingsCard, AmountText, EmptyState, SectionHeader, AppBottomNavigation, PrimaryButton, AppTextField. Semuanya hanya presentasi; perubahan data ada di repository.

Provider: accountRepositoryProvider, accountsProvider, totalBalanceProvider, categoryRepositoryProvider, expenseCategoriesProvider, incomeCategoriesProvider, transactionRepositoryProvider, transactionsProvider, transactionFilterProvider, statisticsProvider, savingRepositoryProvider, savingsProvider, settingsRepositoryProvider, settingsProvider, secureStorageProvider, biometricServiceProvider.

## Keamanan dan backup

`SecureStorageService` dan `BiometricService` mempunyai adapter plugin perangkat asli. Android menggunakan FlutterFragmentActivity, permission biometrik, tema AppCompat, dan backup aplikasi otomatis dinonaktifkan. iOS memiliki penjelasan penggunaan Face ID. Uji adapter pada perangkat fisik sebelum mengaktifkan fitur pengguna; konfigurasi signing/Keychain iOS diverifikasi pada macOS.

`AppLockService` hanya kontrak. Tidak ada PIN yang disimpan, tidak ada lock aktif, dan tidak ada klaim aplikasi sudah terlindungi. Implementasi berikutnya harus memakai salted slow KDF, pembatasan percobaan, proses ganti PIN, pemulihan, dan lifecycle/router guard. Flag biometric di database bukan bukti autentikasi dan tidak boleh dipakai sebagai otorisasi. Database SQLite saat ini tidak dienkripsi.

`BackupService` hanya kontrak export/validasi/restore; belum ada export/import, enkripsi backup atau cloud. Restore harus memvalidasi versi/integritas, mengecualikan secret secure storage, membuat recovery copy, dan mengganti data atomik. Tidak ada login/logout backend dalam aplikasi lokal ini.

## Status

Sudah: struktur feature-first, dark Material 3, 5 tab, seluruh routing placeholder, SQLite persistence/seed, repository dan provider, transaksi atomik, statistik dasar, komponen reusable, adapter keamanan dan kontrak backup.

Belum: UI final, formulir CRUD, detail data, editor filter, scheduler, app lock/PIN, aktivasi biometrik, backup/restore/export, cloud sync, notifikasi, profil pengguna, privacy content, release signing. Lihat `docs/verification.md` untuk hasil pemeriksaan environment.

Referensi implementasi: [Drift setup](https://drift.simonbinder.eu/setup/), [local_auth](https://pub.dev/packages/local_auth), [flutter_secure_storage](https://pub.dev/packages/flutter_secure_storage).
#   S t a r - V a u l t  
 