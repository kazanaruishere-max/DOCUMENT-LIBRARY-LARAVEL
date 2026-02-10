# 🚀 Dokumentasi Laravel Lanjutan (Advanced)

> Panduan ini ditujukan untuk developer yang ingin mendalami fitur-fitur advanced Laravel untuk aplikasi skala besar, API development, dan optimasi performa.

## 📑 Daftar Isi

- [Validation & Form Request](#validation--form-request)
- [Middleware & Kustomisasi](#middleware--kustomisasi)
- [Database Seeding & Factories](#database-seeding--factories)
- [File Storage & Upload](#file-storage--upload)
- [API Development (Sanctum)](#api-development-sanctum)
- [Queue, Jobs & Task Scheduling](#queue-jobs--task-scheduling)
- [Email & Notifications](#email--notifications)
- [Events & Listeners](#events--listeners)
- [Caching (Redis/Memcached)](#caching-redismemcached)
- [Testing (Unit & Feature)](#testing-unit--feature)
- [Deployment & Optimization](#deployment--optimization)

---

## ✅ Validation & Form Request

Untuk menjaga Controller tetap bersih, pindahkan logika validasi ke **Form Request**.

### Membuat Form Request

```bash
php artisan make:request StorePostRequest
```

### Implementasi

```php
// app/Http/Requests/StorePostRequest.php

public function authorize()
{
    return true; // Ubah ke true jika user boleh akses
}

public function rules()
{
    return [
        'title' => 'required|unique:posts|max:255',
        'body' => 'required',
        'publish_at' => 'nullable|date',
    ];
}

public function messages()
{
    return [
        'title.required' => 'Judul wajib diisi',
        'title.unique' => 'Judul sudah ada',
    ];
}
```

### Penggunaan di Controller

```php
public function store(StorePostRequest $request)
{
    // Validasi otomatis dijalankan sebelum masuk method ini
    // Data yang tervalidasi bisa diakses via $request->validated()

    Post::create($request->validated());

    return redirect('/posts');
}
```

---

## 🛡️ Middleware & Kustomisasi

Middleware digunakan untuk memfilter HTTP request yang masuk.

### Membuat Middleware

```bash
php artisan make:middleware CheckAge
```

### Implementasi

```php
// app/Http/Middleware/CheckAge.php

public function handle($request, Closure $next)
{
    if ($request->age <= 200) {
        return redirect('home');
    }

    return $next($request);
}
```

### Register Middleware

Tambahkan di `app/Http/Kernel.php` (Laravel < 11) atau `bootstrap/app.php` (Laravel 11+).

```php
// bootstrap/app.php (Laravel 11)
->withMiddleware(function (Middleware $middleware) {
    $middleware->alias([
        'check.age' => \App\Http\Middleware\CheckAge::class,
    ]);
})
```

### Penggunaan

```php
Route::get('admin/profile', function () {
    //
})->middleware('check.age');
```

---

## 🌱 Database Seeding & Factories

Fitur ini sangat berguna untuk generate dummy data saat development atau testing.

### Membuat Factory

```bash
php artisan make:factory PostFactory
```

### Definisi Factory

```php
// database/factories/PostFactory.php

public function definition()
{
    return [
        'title' => $this->faker->sentence(),
        'body' => $this->faker->paragraph(),
        'user_id' => User::factory(), // Relasi otomatis
        'is_published' => $this->faker->boolean(),
    ];
}
```

### Menjalankan Seeder

```bash
# Isi 50 data dummy
php artisan Tinker
>>> \App\Models\Post::factory()->count(50)->create();

# Atau via Seeder Class
php artisan db:seed
```

---

## 📂 File Storage & Upload

Laravel menyediakan abstraksi filesystem yang powerful (Local, S3, FTP).

### Konfigurasi

Setting di `config/filesystems.php` atau `.env`.

### Upload File

```php
// Upload ke local storage (storage/app/public)
if ($request->hasFile('avatar')) {
    $path = $request->file('avatar')->store('avatars', 'public');

    // Simpan path ke database
    $user->avatar = $path;
    $user->save();
}

// Upload ke S3
$path = $request->file('avatar')->store('avatars', 's3');
```

### Akses File

Pastikan symlink sudah dibuat: `php artisan storage:link`

```blade
<img src="{{ Storage::url($user->avatar) }}" alt="Avatar">
```

---

## 🔌 API Development (Sanctum)

Sanctum adalah solusi ringan untuk authentication API (SPA, Mobile App).

### Instalasi

Laravel baru sudah include Sanctum. Jika belum:

```bash
php artisan install:api
```

### Proteksi Route API

```php
// routes/api.php

Route::middleware('auth:sanctum')->get('/user', function (Request $request) {
    return $request->user();
});
```

### Login API & Generate Token

```php
public function login(Request $request)
{
    $request->validate([
        'email' => 'required|email',
        'password' => 'required',
    ]);

    $user = User::where('email', $request->email)->first();

    if (! $user || ! Hash::check($request->password, $user->password)) {
        return response()->json(['message' => 'Unauthorized'], 401);
    }

    // Generate Token
    $token = $user->createToken('auth_token')->plainTextToken;

    return response()->json([
        'message' => 'Login success',
        'access_token' => $token,
        'token_type' => 'Bearer'
    ]);
}
```

---

## ⏳ Queue, Jobs & Task Scheduling

Pindahkan proses berat (kirim email, generate PDF, proses gambar) ke background job.

### Membuat Job

```bash
php artisan make:job ProcessPodcast
```

### Implementasi Job

```php
// app/Jobs/ProcessPodcast.php

public function handle()
{
    // Proses berat di sini
    // Misalnya encode video
}
```

### Dispatch Job

```php
ProcessPodcast::dispatch($podcast);
```

### Menjalankan Worker

```bash
php artisan queue:work
```

### Task Scheduling (Cron Job)

Di `routes/console.php` :

```php
use Illuminate\Support\Facades\Schedule;

Schedule::command('emails:send')->daily();
Schedule::job(new DeleteRecentUsers)->dailyAt('01:00');
```

---

## 📧 Email & Notifications

### Membuat Mailable

```bash
php artisan make:mail OrderShipped
```

### Kirim Email

```php
use App\Mail\OrderShipped;
use Illuminate\Support\Facades\Mail;

Mail::to($request->user())->send(new OrderShipped($order));
```

### Notifications (Email, Slack, SMS, Database)

```bash
php artisan make:notification InvoicePaid
```

```php
// Kirim notifikasi
$user->notify(new InvoicePaid($invoice));
```

---

## 📡 Events & Listeners

Pola Observer pattern untuk decoupling kode. Misal: Saat user register, kirim email welcome + log activity.

### Membuat Event & Listener

```bash
php artisan make:event UserRegistered
php artisan make:listener SendWelcomeEmail --event=UserRegistered
```

### Implementasi

1.  **Event**: Menerima data
2.  **Listener**: Melakukan aksi
3.  **Dispatch**:

```php
UserRegistered::dispatch($user);
```

---

## ⚡ Caching (Redis/Memcached)

Caching meningkatkan performa drastis dengan menyimpan hasil query/proses di memori.

### Penggunaan Cache

```php
use Illuminate\Support\Facades\Cache;

// Simpan data selama 60 menit (3600 detik)
$users = Cache::remember('users', 3600, function () {
    return DB::table('users')->get();
});

// Cache selamanya
Cache::forever('key', 'value');

// Hapus cache
Cache::forget('key');
```

---

## 🧪 Testing (Unit & Feature)

Laravel dibuat dengan mindset testing-first.

### Membuat Test

```bash
php artisan make:test UserTest
```

### Contoh Feature Test

```php
public function test_user_can_view_login_page()
{
    $response = $this->get('/login');

    $response->assertStatus(200);
}

public function test_user_can_login()
{
    $user = User::factory()->create();

    $response = $this->post('/login', [
        'email' => $user->email,
        'password' => 'password',
    ]);

    $this->assertAuthenticated();
    $response->assertRedirect('/home');
}
```

### Menjalankan Test

```bash
php artisan test
```

---

## 🚀 Deployment & Optimization

### Optimization Commands (Production Only)

```bash
# Install dependency tanpa dev
composer install --optimize-autoloader --no-dev

# Cache Config, Events, Routes, Views
php artisan config:cache
php artisan event:cache
php artisan route:cache
php artisan view:cache
```

### Environment

Pastikan `.env` di server production:

- `APP_ENV=production`
- `APP_DEBUG=false`

### Server Configuration (Nginx)

Pastikan root mengarah ke folder `public`.

```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/example.com/public;

    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }
}
```

---

**Selamat! Anda sudah menguasai fitur-fitur "Sakti" Laravel.** 🧙‍♂️
Gunakan fitur-fitur ini untuk membangun aplikasi yang scalable, maintainable, dan high-performance.
