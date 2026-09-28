# Deploy iOS & Android (GitLab → GitHub Actions → TestFlight / Google Play)

Tài liệu hướng dẫn từ A tới Z cách setup và vận hành pipeline deploy app lên TestFlight và Google Play, tận dụng runner macOS miễn phí của GitHub Actions trong khi source code vẫn nằm trên GitLab.

## Mục lục

1. [Tổng quan](#1-tổng-quan)
2. [Chi phí & giới hạn](#2-chi-phí--giới-hạn)
3. [Chuẩn bị](#3-chuẩn-bị)
4. [Đổi bundle ID / package name](#4-đổi-bundle-id--package-name)
5. [Setup Apple (iOS)](#5-setup-apple-ios)
6. [Setup Google Play (Android)](#6-setup-google-play-android)
7. [Setup repo GitHub](#7-setup-repo-github)
8. [Setup GitLab](#8-setup-gitlab)
9. [Chạy lần đầu](#9-chạy-lần-đầu)
10. [Deploy hằng ngày](#10-deploy-hằng-ngày)
11. [Sau khi upload](#11-sau-khi-upload)
12. [Xử lý lỗi thường gặp](#12-xử-lý-lỗi-thường-gặp)
13. [Bảo mật & bảo trì](#13-bảo-mật--bảo-trì)
14. [Phụ lục: không dùng API key (.p8)](#14-phụ-lục-không-dùng-api-key-p8)
15. [Phụ lục: so sánh với workflow mẫu (DGF Connect Admin)](#15-phụ-lục-so-sánh-với-workflow-mẫu-dgf-connect-admin)

---

## 1. Tổng quan

```
GitLab (laboro-app)                         GitHub (repo CI riêng)
───────────────────                         ──────────────────────
push develop/main
  └─ verify (format, analyze, test)
  └─ release:dev / release:prod  (bấm tay)
       └─ tạo tag v1.0.0+87-prod
            └─ pipeline của tag
                 └─ verify
                 └─ deploy ── repository_dispatch ──▶  Build & Deploy
                                                         ├─ params  (đọc tag)
                                                         ├─ android (ubuntu)  ─▶ Google Play
                                                         └─ ios     (macos)   ─▶ TestFlight
```

- **GitLab** giữ source code, chạy verify, tạo tag và gọi sang GitHub.
- **GitHub** chỉ chứa **1 file** `.github/workflows/deploy.yml`. Workflow clone source từ GitLab theo tag rồi build + upload. Source code không nằm lại trên GitHub (runner bị xoá sau mỗi job).
- Fastfile và `ExportOptions.plist` cho iOS được workflow **tự sinh lúc chạy**, repo laboro không cần chứa fastlane.
- Chứng chỉ iOS quản lý bằng **fastlane match**, lưu mã hoá trong một repo git private riêng.

### Quy ước tag

```
v<version>+<build_number>-<flavor>
v1.0.0+87-prod
```

| Phần | Ý nghĩa |
| --- | --- |
| `version` | Lấy từ `version:` trong `pubspec.yaml` (phần trước dấu `+`) |
| `build_number` | Số pipeline GitLab (`CI_PIPELINE_IID`), luôn tăng, dùng làm build number iOS / versionCode Android |
| `flavor` | `dev` → dùng `config/development.json`, `prod` → dùng `config/production.json` |

## 2. Chi phí & giới hạn

| Hạng mục | Chi phí |
| --- | --- |
| Apple Developer Program | **$99/năm** (bắt buộc, không có cách miễn phí) |
| Google Play Console | **$25** một lần |
| GitHub Actions, repo **public** | Miễn phí không giới hạn phút (log build ai cũng xem được) |
| GitHub Actions, repo **private** | 2.000 phút/tháng, **macOS tính x10** → ~200 phút macOS ≈ 8–12 lần build iOS/tháng. Linux (Android) tính x1 |
| GitLab CI | Dùng runner Linux như hiện tại, các job trigger chỉ tốn vài giây |

Workflow có `timeout-minutes` để một build bị treo không ăn hết quota. Chỉ deploy khi cần (bấm tay `release:*`), không tự deploy mỗi lần push.

## 3. Chuẩn bị

Tài khoản:

- Apple Developer Program (quyền **Admin** hoặc **Account Holder** để tạo API key).
- Google Play Console (quyền Admin của app).
- Google Cloud project (để tạo service account).
- Tài khoản GitHub (tạo repo CI).
- Quyền **Maintainer** trên project GitLab `laboro-app` (để thêm CI/CD variables và tạo token).

Công cụ trên máy (Linux/macOS đều được):

- `keytool` (có sẵn khi cài JDK) — tạo keystore Android.
- `base64` — encode file vào secret.
- Không cần Mac: việc tạo chứng chỉ iOS chạy trên runner macOS của GitHub.

## 4. Đổi bundle ID / package name

Apple và Google **từ chối** mọi ID bắt đầu bằng `com.example`. Hiện project vẫn đang là:

| Nền tảng | Giá trị hiện tại | File |
| --- | --- | --- |
| iOS | `com.example.laboroWorkforceMobile` | `ios/Runner.xcodeproj/project.pbxproj` (`PRODUCT_BUNDLE_IDENTIFIER`, cả `Runner` và `RunnerTests`) |
| Android | `com.example.laboro_workforce_mobile` | `android/app/build.gradle.kts` (`applicationId` và `namespace`) + package của `MainActivity.kt` |

Chọn ID chính thức (ví dụ `com.vistarsoft.laboro`), sửa trong repo, commit. Sau đó cập nhật 2 biến ở đầu workflow:

```yaml
env:
  APPLE_TEAM_ID: HNVG2MT2JL
  IOS_BUNDLE_ID: com.vistarsoft.laboro
  ANDROID_PACKAGE_NAME: com.vistarsoft.laboro
```

> ID này **không đổi được** sau khi đã phát hành, hãy chốt kỹ trước.

## 5. Setup Apple (iOS)

### 5.1 Tạo App ID

1. Vào [developer.apple.com/account](https://developer.apple.com/account) → **Certificates, Identifiers & Profiles** → **Identifiers** → **+**.
2. Chọn **App IDs** → **App** → nhập Description và **Bundle ID (Explicit)** = bundle ID ở bước 4.
3. Tick các Capabilities app cần (Push Notifications, Sign in with Apple…) → **Register**.

### 5.2 Tạo app trên App Store Connect

1. [appstoreconnect.apple.com](https://appstoreconnect.apple.com) → **Apps** → **+** → **New App**.
2. Platform **iOS**, chọn Bundle ID vừa tạo, SKU tuỳ ý (vd `laboro-ios`).

### 5.3 Tạo App Store Connect API key (.p8)

API key giúp CI đăng nhập Apple **không cần 2FA**.

1. App Store Connect → **Users and Access** → **Integrations** → **App Store Connect API** → **Team Keys** → **+**.
2. Tên: `GitHub CI`, Access: **App Manager** → **Generate**.
3. Ghi lại:
   - **Issuer ID** (hiển thị phía trên danh sách key).
   - **Key ID** (cột Key ID).
4. Bấm **Download API Key** → được file `AuthKey_XXXXXXXXXX.p8`.

> File `.p8` **chỉ tải được một lần**. Mất thì phải revoke và tạo key mới.

Encode để đưa vào secret:

```bash
base64 -w0 AuthKey_XXXXXXXXXX.p8        # Linux
base64 -i AuthKey_XXXXXXXXXX.p8 | tr -d '\n'   # macOS
```

### 5.4 Tạo repo lưu chứng chỉ (match)

1. Tạo một repo **private, rỗng** (GitLab hoặc GitHub đều được), ví dụ `laboro-ios-certs`.
2. Tạo token có quyền **đọc + ghi** repo đó:
   - GitLab: **Project access token** (role Developer, scope `read_repository`, `write_repository`) hoặc Personal access token.
   - GitHub: Fine-grained PAT, quyền **Contents: Read and write** trên repo certs.
3. Tạo chuỗi basic auth:

```bash
echo -n "<username>:<token>" | base64 -w0
```

   Với GitLab project access token, `<username>` có thể là bất kỳ chuỗi nào không rỗng (vd `oauth2`).

4. Tự đặt một **MATCH_PASSWORD** mạnh (dùng để mã hoá chứng chỉ trong repo). Lưu vào password manager — mất là phải tạo lại toàn bộ chứng chỉ.

## 6. Setup Google Play (Android)

### 6.1 Tạo upload keystore

```bash
keytool -genkeypair -v \
  -keystore upload-keystore.jks \
  -keyalg RSA -keysize 2048 -validity 10000 \
  -alias upload
```

Ghi lại **store password**, **key alias** (`upload`), **key password**. Encode:

```bash
base64 -w0 upload-keystore.jks
```

> Lưu file `.jks` và mật khẩu ở nơi an toàn (password manager / vault). **Không commit vào repo.**

Workflow ký AAB bằng các thuộc tính `android.injected.signing.*` ghi vào `~/.gradle/gradle.properties` của runner, nên **không cần sửa** `android/app/build.gradle.kts`.

### 6.2 Tạo app trên Play Console

1. [play.google.com/console](https://play.google.com/console) → **Create app** → điền tên, ngôn ngữ, loại app, free/paid.
2. Hoàn thành các mục bắt buộc ở **Dashboard → Set up your app** (privacy policy, content rating, target audience, data safety…).
3. Bật **Play App Signing** (mặc định bật cho app mới). Keystore ở 6.1 là **upload key**, Google giữ app signing key.

### 6.3 Upload bản đầu tiên bằng tay

Google Play API **không cho upload** vào app chưa có bản build nào. Build một AAB đã ký bằng upload keystore trên máy (dùng cùng cơ chế với CI, không cần sửa Gradle):

```bash
cat >> ~/.gradle/gradle.properties <<EOF
android.injected.signing.store.file=/đường/dẫn/upload-keystore.jks
android.injected.signing.store.password=<store password>
android.injected.signing.key.alias=upload
android.injected.signing.key.password=<key password>
EOF

fvm flutter build appbundle --release --dart-define-from-file=config/development.json
```

Build xong nhớ xoá 4 dòng trên khỏi `~/.gradle/gradle.properties`, nếu không mọi bản release build trên máy sẽ bị ký bằng upload key. Sau đó vào **Testing → Internal testing → Create new release** → upload `build/app/outputs/bundle/release/app-release.aab`.

### 6.4 Tạo service account

1. [console.cloud.google.com](https://console.cloud.google.com) → chọn/tạo project → **APIs & Services** → **Enable APIs** → bật **Google Play Android Developer API**.
2. **IAM & Admin → Service Accounts → Create service account**, tên `play-deploy`, không cần gán role GCP.
3. Mở service account → **Keys → Add key → Create new key → JSON** → tải file JSON.
4. Play Console → **Users and permissions** → **Invite new users** → dán email của service account (`...@...iam.gserviceaccount.com`).
5. Tab **App permissions** → thêm app laboro → cấp tối thiểu:
   - Release apps to testing tracks
   - Release to production, exclude devices, and use Play App Signing (nếu muốn deploy production)
   - Manage testing tracks and edit tester lists
6. **Invite user**. Quyền có thể mất vài phút tới vài giờ mới có hiệu lực.

### 6.5 App chưa từng được publish

Khi app còn ở trạng thái **draft** (chưa phát hành bản nào), Google chỉ chấp nhận release có `status: draft`. Trong workflow, sửa tạm:

```yaml
      - name: Upload to Google Play
        uses: r0adkll/upload-google-play@v1
        with:
          ...
          status: draft
```

Sau khi app đã được duyệt và phát hành lần đầu, đổi lại `completed`.

## 7. Setup repo GitHub

### 7.1 Tạo repo

1. Tạo repo mới trên GitHub, ví dụ `laboro-ci`.
   - **Public**: không giới hạn phút macOS, nhưng log build công khai (secret vẫn bị che `***`).
   - **Private**: ~200 phút macOS/tháng.
2. Đưa file workflow vào **đúng đường dẫn** `.github/workflows/deploy.yml`:

```bash
mkdir -p laboro-ci/.github/workflows
cp ~/workspace/github-ci/workflows/deploy.yml laboro-ci/.github/workflows/deploy.yml
cd laboro-ci
git init -b main
git add .
git commit -m "Add deploy workflow"
git remote add origin git@github.com:<owner>/laboro-ci.git
git push -u origin main
```

> `repository_dispatch` chỉ chạy workflow nằm trên **default branch** (`main`). Sửa workflow ở nhánh khác sẽ không có tác dụng.

### 7.2 Secrets

Repo GitHub → **Settings → Secrets and variables → Actions → New repository secret**.

> Secret gắn với **từng repo**. Không dùng chung repo CI với app khác, vì keystore, service account và env file là của riêng từng app.

**Chung**

| Secret | Giá trị | Lấy từ đâu |
| --- | --- | --- |
| `GITLAB_REPO_PATH` | `vistarsoftware/vistarsoft-laboro/laboro-app` | Đường dẫn project GitLab |
| `GITLAB_DEPLOY_TOKEN` | Token đọc repo GitLab | GitLab → project → **Settings → Access tokens** → Project access token, role **Reporter**, scope `read_repository`. (Không dùng *Deploy token* vì workflow đăng nhập với username `gitlab-ci-token`) |
| `ENV_FILE_DEV` | Nội dung JSON cho flavor `dev` | Giống format `config/development.json`. Để trống thì dùng file `config/development.json` đã commit |
| `ENV_FILE_PROD` | Nội dung JSON cho flavor `prod` | Giống format `config/production.example.json`, **bắt buộc** |

Ví dụ `ENV_FILE_PROD`:

```json
{
  "APP_ENV": "production",
  "API_BASE_URL": "https://api.laboro.vn/v1/"
}
```

> Phải là **JSON**, không phải định dạng `.env` (`KEY=value`).

**Android**

| Secret | Giá trị |
| --- | --- |
| `ANDROID_KEYSTORE_BASE64` | Kết quả `base64 -w0 upload-keystore.jks` (mục 6.1) |
| `ANDROID_KEYSTORE_PASSWORD` | Store password |
| `ANDROID_KEY_ALIAS` | Key alias (`upload`) |
| `ANDROID_KEY_PASSWORD` | Key password |
| `GOOGLE_PLAY_SERVICE_ACCOUNT_JSON` | **Toàn bộ nội dung** file JSON service account (mục 6.4), dán thẳng, không base64 |

**iOS**

| Secret | Giá trị |
| --- | --- |
| `APP_STORE_ISSUER_ID` | Issuer ID (mục 5.3) |
| `APP_STORE_KEY_ID` | Key ID (mục 5.3) |
| `APP_STORE_KEY_P8_BASE64` | File `.p8` đã base64 (mục 5.3) |
| `MATCH_GIT_URL` | URL **HTTPS** repo certs, vd `https://gitlab.com/vistarsoftware/laboro-ios-certs.git` |
| `MATCH_GIT_BASIC_AUTHORIZATION` | Chuỗi base64 `username:token` (mục 5.4) |
| `MATCH_PASSWORD` | Mật khẩu mã hoá chứng chỉ (mục 5.4) |

### 7.3 Workflow làm gì

| Job | Runner | Các bước |
| --- | --- | --- |
| `params` | ubuntu | Đọc tag → `version`, `build_number`, `flavor`; kiểm tra định dạng tag |
| `android` | ubuntu | Clone tag từ GitLab → cài Flutter theo `.fvmrc` → ghi config → `pub get`, `gen-l10n`, `build_runner` → giải mã keystore → `flutter build appbundle` → upload Google Play |
| `ios` | macos-26 | Chọn Xcode stable mới nhất → clone tag → sinh Fastfile + ExportOptions.plist → cài fastlane → `fastlane ios prepare` (match + chuyển project sang manual signing) → `flutter build ipa` → `fastlane ios upload` (TestFlight) |

## 8. Setup GitLab

### 8.1 CI/CD variables

GitLab project → **Settings → CI/CD → Variables → Add variable**. Tick **Mask variable**.

| Key | Giá trị | Dùng để |
| --- | --- | --- |
| `GITHUB_TOKEN` | GitHub fine-grained PAT: **Repository access** = chỉ repo `laboro-ci`, **Permissions → Contents: Read and write** | Job `deploy` gọi `repository_dispatch` |
| `GITLAB_PUSH_TOKEN` | GitLab Project access token, role **Maintainer** (hoặc Developer nếu tag không bị protect), scope **`api`** | Job `release:*` tạo tag qua API |

> Nếu tag `v*` được cấu hình **Protected tags**, token tạo tag cần role được phép tạo protected tag.

### 8.2 `.gitlab-ci.yml`

Các phần đã có trong `.gitlab-ci.yml`:

```yaml
variables:
  GITHUB_CI_REPO: "<github-owner>/<github-repo>"   # ← sửa thành repo thật, vd vistarsoft/laboro-ci
  DEPLOY_PLATFORM: "both"

stages:
  - verify
  - build
  - release
```

| Job | Chạy khi | Làm gì |
| --- | --- | --- |
| `release:dev` | Pipeline nhánh `develop`, **bấm tay** | Tạo tag `v<version>+<pipeline_iid>-dev` tại commit hiện tại |
| `release:prod` | Pipeline nhánh `main`, **bấm tay** | Tạo tag `v<version>+<pipeline_iid>-prod` |
| `deploy` | Pipeline của tag khớp `v*.*.*+N-(dev\|prod)` | Gọi GitHub `repository_dispatch` với tag + platform |

Muốn cho phép release từ nhánh khác, sửa điều kiện `rules` của `release:dev` / `release:prod`.

## 9. Chạy lần đầu

Làm theo đúng thứ tự:

1. **Kiểm tra checklist**
   - [ ] Đã đổi bundle ID / package name (mục 4) và cập nhật `env` trong workflow.
   - [ ] App ID + app trên App Store Connect đã tạo (5.1, 5.2).
   - [ ] App trên Play Console đã tạo và có bản upload tay đầu tiên (6.2, 6.3).
   - [ ] Đủ secrets GitHub (7.2) và variables GitLab (8.1).
   - [ ] Đã sửa `GITHUB_CI_REPO` trong `.gitlab-ci.yml`.

2. **Tạo chứng chỉ iOS (chỉ một lần)**
   - GitHub → repo `laboro-ci` → **Actions** → **Build & Deploy** → **Run workflow**.
   - Bật **init_certs**, các ô khác để mặc định → **Run workflow**.
   - Job `ios` sẽ tạo Apple Distribution certificate + App Store provisioning profile rồi đẩy (đã mã hoá) vào repo certs.
   - Kiểm tra: repo certs có thư mục `certs/distribution` và `profiles/appstore`.

3. **Thử deploy bản dev**
   - GitLab → **Build → Pipelines** → pipeline mới nhất của `develop` → bấm ▶ **`release:dev`**.
   - Một pipeline mới cho tag xuất hiện → chờ `verify` và `deploy` xong.
   - Log `deploy` in link tới GitHub Actions → theo dõi 2 job `android` và `ios`.

4. **Kiểm tra kết quả**
   - TestFlight: build xuất hiện ở App Store Connect → **TestFlight** sau khoảng 5–20 phút xử lý.
   - Google Play: **Testing → Internal testing** có release mới.

## 10. Deploy hằng ngày

### Cách thông thường

1. Tăng `version:` trong `pubspec.yaml` nếu là bản mới (phần build number sau `+` không quan trọng, CI tự đặt).
2. Merge vào `develop` (bản dev) hoặc `main` (bản prod).
3. Mở pipeline của nhánh → bấm **`release:dev`** hoặc **`release:prod`**.

### Chỉ deploy một nền tảng

GitLab → **Build → Pipelines → Run pipeline** → chọn **tag** cần deploy (đã tạo từ `release:*`) → thêm biến:

| Key | Value |
| --- | --- |
| `DEPLOY_PLATFORM` | `ios` hoặc `android` |

Hoặc chạy trực tiếp trên GitHub: **Actions → Build & Deploy → Run workflow**, nhập `tag` (đã tồn tại trên GitLab) và chọn `platform`.

### Đổi track Android

Mặc định upload vào track `internal`. Chạy bằng tay trên GitHub và chọn `android_track` = `alpha` / `beta` / `production`.

### Deploy lại một tag cũ

Không tạo tag mới, chạy workflow tay trên GitHub với tag đó. Lưu ý: TestFlight và Play **không chấp nhận build number trùng**, nên chỉ deploy lại được khi lần trước upload thất bại.

## 11. Sau khi upload

**iOS (TestFlight)**

- Lần đầu, App Store Connect yêu cầu khai **Export Compliance**. Nếu app chỉ dùng HTTPS tiêu chuẩn, thêm vào `ios/Runner/Info.plist` để khỏi hỏi lại mỗi build:

  ```xml
  <key>ITSAppUsesNonExemptEncryption</key>
  <false/>
  ```

- **TestFlight → Internal Testing** → tạo group, thêm tester (thành viên team App Store Connect). External testing cần Apple review bản đầu.
- Phát hành App Store: tạo version mới trên App Store Connect → chọn build → **Submit for Review**.

**Android (Google Play)**

- Internal testing: thêm email tester ở **Testing → Internal testing → Testers**, gửi link opt-in.
- Promote lên production: **Internal testing → Promote release → Production**, hoặc deploy với `android_track: production`.

## 12. Xử lý lỗi thường gặp

| Triệu chứng | Nguyên nhân | Cách xử lý |
| --- | --- | --- |
| Job `deploy` (GitLab) trả `401` / `403` | `GITHUB_TOKEN` hết hạn hoặc thiếu quyền Contents write | Tạo lại PAT, đúng repo, quyền Contents: Read and write |
| Job `deploy` trả `404` | Sai `GITHUB_CI_REPO` hoặc token không có quyền với repo | Kiểm tra `owner/repo` |
| `deploy` thành công nhưng GitHub không chạy | Workflow không nằm ở `.github/workflows/` trên nhánh `main` | Kiểm tra đường dẫn file và default branch |
| `release:*` trả `401` / `403` | `GITLAB_PUSH_TOKEN` thiếu scope `api` hoặc không được tạo protected tag | Cấp lại token |
| `release:*` trả `400 Tag ... already exists` | Chạy lại job trong cùng pipeline | Chạy pipeline mới (build number = pipeline IID) |
| `Tag không hợp lệ` ở job `params` | Tag sai định dạng | Đúng dạng `v1.2.0+15-prod` |
| Clone GitLab lỗi `Authentication failed` | `GITLAB_DEPLOY_TOKEN` hết hạn / thiếu `read_repository` / là deploy token | Tạo Project access token mới |
| `Thiếu config/production.json` | `ENV_FILE_PROD` trống | Thêm secret |
| `FormatException` khi build | `ENV_FILE_*` không phải JSON hợp lệ | Kiểm tra bằng `jq . <file>` trước khi dán |
| match: `Could not find ... certificate` / `No matching provisioning profiles` | Chưa chạy `init_certs`, hoặc đổi bundle ID sau khi tạo cert | Chạy lại workflow với `init_certs` |
| match: `Invalid password` | `MATCH_PASSWORD` sai | Dùng đúng mật khẩu lúc tạo; mất thì xoá repo certs, revoke cert trên Apple, chạy lại `init_certs` |
| match: lỗi clone repo certs | `MATCH_GIT_URL` không phải HTTPS hoặc `MATCH_GIT_BASIC_AUTHORIZATION` sai | Encode lại `username:token` |
| `You have already reached the maximum allowed number of certificates` | Đã có quá nhiều Distribution certificate | Revoke cert cũ không dùng trên developer.apple.com rồi chạy `init_certs` |
| `The bundle version must be higher than the previously uploaded version` | Build number trùng | Tạo tag mới qua `release:*` |
| `Authentication credentials are missing or invalid` (App Store) | Sai Key ID / Issuer ID / p8, hoặc key đã bị revoke | Kiểm tra 3 secret `APP_STORE_*` |
| Play: `Package not found: ...` | Chưa upload tay bản đầu, hoặc sai `ANDROID_PACKAGE_NAME` | Mục 6.3 |
| Play: `Only releases with status draft may be created on draft app` | App chưa từng publish | Mục 6.5 |
| Play: `The caller does not have permission` | Service account chưa được mời hoặc thiếu quyền release | Mục 6.4, chờ quyền có hiệu lực |
| Play: `Version code N has already been used` | Build number trùng | Tạo tag mới |
| Play: `APK signed with the wrong key` | Keystore khác upload key đã đăng ký | Dùng đúng keystore; nếu mất, yêu cầu reset upload key trong Play Console |
| Job iOS bị huỷ vì timeout | Build quá lâu | Tăng `timeout-minutes` hoặc kiểm tra bước bị treo |
| Hết phút GitHub Actions | Repo private, hết ~200 phút macOS | Chờ tháng sau, chuyển repo CI sang public, hoặc mua thêm |

Xem log chi tiết: GitHub → **Actions** → run tương ứng → bấm vào từng step.

## 13. Bảo mật & bảo trì

- **Không commit** `.p8`, `.jks`, `.p12`, JSON service account hay mật khẩu vào bất kỳ repo nào. Mọi thứ nằm trong GitHub Secrets / GitLab Variables.
- Lưu bản gốc của keystore, `.p8`, `MATCH_PASSWORD` trong password manager của team.
- Token có hạn dùng: ghi lại ngày hết hạn của `GITHUB_TOKEN`, `GITLAB_PUSH_TOKEN`, `GITLAB_DEPLOY_TOKEN`, token repo certs và gia hạn trước.
- Khi một thành viên nắm secret rời team: revoke API key `.p8` và tạo key mới, xoay các token.
- Apple Distribution certificate hết hạn sau **1 năm**: khi hết hạn, xoá cert/profile cũ trong repo certs (`fastlane match nuke distribution` trên Mac hoặc xoá tay) rồi chạy `init_certs`.
- Apple định kỳ nâng yêu cầu Xcode/SDK tối thiểu. Workflow dùng `xcode-version: latest-stable` và `runs-on: macos-26`; khi GitHub ra image mới, cập nhật `runs-on`.
- Nâng Flutter: chỉ cần sửa `.fvmrc`, workflow tự đọc phiên bản.
- File workflow gốc được lưu ở `~/workspace/github-ci/workflows/deploy.yml`; sửa ở đó rồi copy sang repo GitHub (hoặc biến thư mục đó thành chính repo `laboro-ci`).

## 14. Phụ lục: không dùng API key (.p8)

Nếu không muốn dùng `.p8`, có thể thay bằng **Apple ID + app-specific password**. Nhược điểm: phải có **máy Mac** để tạo chứng chỉ lần đầu (đăng nhập 2FA), và tài khoản cá nhân gắn với CI.

1. Trên Mac, cài fastlane (`brew install fastlane`), rồi trong thư mục `ios/`:

   ```bash
   export MATCH_GIT_URL=https://gitlab.com/vistarsoftware/laboro-ios-certs.git
   export MATCH_PASSWORD=<mật khẩu>
   fastlane match appstore \
     --app_identifier com.vistarsoft.laboro \
     --team_id HNVG2MT2JL
   ```

   Đăng nhập Apple ID và nhập mã 2FA khi được hỏi. Bước này thay cho `init_certs`.

2. Tạo app-specific password tại [account.apple.com](https://account.apple.com) → **Sign-In and Security → App-Specific Passwords**.

3. Thêm GitHub secrets `FASTLANE_USER` (Apple ID email) và `FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD`; bỏ 3 secret `APP_STORE_*`.

4. Sửa workflow:
   - Env job `ios`: thay 3 dòng `ASC_*` bằng

     ```yaml
     FASTLANE_USER: ${{ secrets.FASTLANE_USER }}
     FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD: ${{ secrets.FASTLANE_APPLE_APPLICATION_SPECIFIC_PASSWORD }}
     ```

   - Trong Fastfile sinh ra: xoá lane `asc_key`, bỏ `api_key: asc_key` ở `match` (readonly không cần đăng nhập Apple), và lane `upload` đổi thành:

     ```ruby
     upload_to_testflight(
       ipa: Dir["../../build/ios/ipa/*.ipa"].first,
       skip_waiting_for_build_processing: true,
       apple_id: "<Apple ID số của app, xem ở App Store Connect → App Information>"
     )
     ```

   - Bỏ bước `Init certificates` / input `init_certs` (đã làm ở bước 1 trên Mac).

## 15. Phụ lục: so sánh với workflow mẫu (DGF Connect Admin)

Workflow của laboro được xây dựng dựa trên workflow CI của project **DGF Connect Admin** (repo public `dgf-connect-admin-ci`). Mục này ghi lại những điểm giống/khác và lý do, để khi chỉnh sửa có thể tham khảo.

### 15.1 Điểm giống nhau

- Mô hình: source trên GitLab, repo GitHub riêng chỉ chứa workflow, trigger bằng `repository_dispatch` (`event_type: deploy`).
- Runner clone source từ GitLab theo tag bằng `gitlab-ci-token:${GITLAB_DEPLOY_TOKEN}` và `GITLAB_REPO_PATH`.
- Job Android chạy trên `ubuntu-latest`, job iOS chạy trên `macos-26`.
- Dùng chung tên secret: `ANDROID_KEYSTORE_BASE64`, `ANDROID_KEYSTORE_PASSWORD`, `ANDROID_KEY_ALIAS`, `ANDROID_KEY_PASSWORD`, `GOOGLE_PLAY_SERVICE_ACCOUNT_JSON`, `APP_STORE_ISSUER_ID`, `ENV_FILE_DEV`, `ENV_FILE_PROD`, `GITLAB_DEPLOY_TOKEN`, `GITLAB_REPO_PATH`.
- Payload có `tag` và `platform` (`ios` | `android` | `both`); tag có hậu tố flavor `-dev` / `-prod`.

### 15.2 Điểm khác nhau

Khác biệt cốt lõi: **DGF để logic build/ký/upload trong source GitLab** (Fastfile, Gemfile, ExportOptions plist nằm trong repo app), workflow chỉ gọi lane. **Laboro để toàn bộ logic trong workflow** (tự sinh Fastfile và ExportOptions.plist lúc chạy) vì repo laboro chưa có fastlane.

| Hạng mục | DGF Connect Admin | Laboro | Lý do |
| --- | --- | --- | --- |
| Trigger | Chỉ `repository_dispatch` (từ `deploy.sh` local hoặc GitLab CI) | `repository_dispatch` + `workflow_dispatch` (nút Run workflow) | Cần chạy tay để tạo chứng chỉ lần đầu (`init_certs`), deploy lại tag, đổi track. Trên repo public chỉ người có quyền write mới bấm được |
| Payload | `tag`, `flavor`, `version`, `build_number`, `platform`; thiếu thì tách từ tag | `tag`, `platform`, `android_track`; version / build number / flavor **luôn** tách từ tag | Một nguồn sự thật duy nhất là tag, tránh lệch giữa tag và payload |
| Kiểm tra tag | Không | Regex `^v\d+\.\d+\.\d+\+\d+-(dev\|prod)$`, sai là dừng | Bắt lỗi sớm thay vì build hỏng giữa chừng |
| Parse tag | Lặp lại trong 2 job | Job `params` riêng, 2 job dùng chung output | Tránh trùng lặp |
| Flavor | Flavor thật: `--flavor`, scheme riêng, `ExportOptions_<flavor>.plist` | Không có flavor native; `dev` → `config/development.json`, `prod` → `config/production.json` | Laboro chưa cấu hình flavor/scheme; phân môi trường bằng `--dart-define-from-file` |
| Env file | `.env` → `assets/env/.env.<flavor>` | **JSON** → `config/<env>.json`; `ENV_FILE_DEV` trống thì dùng file đã commit | Theo cấu trúc `config/` sẵn có của laboro |
| Flutter version | Viết cứng `3.41.7` | Đọc từ `.fvmrc` | Không bị lệch với phiên bản dev dùng local |
| Clone | Full clone rồi `git checkout <tag>` | `git clone --depth 1 --branch <tag>` | Nhanh hơn |
| Fastlane | Fastfile + Gemfile trong source, `bundle exec` | Fastfile sinh trong workflow, `gem install fastlane` | Laboro không có fastlane trong repo |
| `flutter pub run` | Có (đã deprecated) | `dart run` | Lệnh mới |
| `gen-l10n` | Không | Có | Laboro dùng l10n sinh code |
| Android build | Lane `android deploy_<flavor>` trong repo | `flutter build appbundle` + ký qua `android.injected.signing.*` trong `~/.gradle/gradle.properties` | Không cần sửa `build.gradle.kts` |
| Android upload | Lane fastlane (`upload_to_play_store`) | Action `r0adkll/upload-google-play@v1` | Không cần fastlane cho Android |
| Android track | Cố định trong Fastfile | Mặc định `internal`, chọn được qua `android_track` | Linh hoạt khi promote |
| Ký iOS | Lane `prepare_<flavor>` trong repo; credential **không nằm trong GitHub secrets** (nhiều khả năng commit trong source) | `match` readonly, chứng chỉ trong repo certs riêng, credential trong secrets | Không để key/cert trong source code |
| API key App Store | Chỉ `APP_STORE_ISSUER_ID` trong secrets (Key ID + `.p8` nằm trong source) | `APP_STORE_ISSUER_ID` + `APP_STORE_KEY_ID` + `APP_STORE_KEY_P8_BASE64` | Như trên |
| Secrets match | Không có | `MATCH_GIT_URL`, `MATCH_GIT_BASIC_AUTHORIZATION`, `MATCH_PASSWORD` | Match cần truy cập và giải mã repo certs |
| Tạo chứng chỉ lần đầu | Ngoài CI | Input `init_certs` chạy `match` (readonly: false) trên runner macOS | Không cần máy Mac |
| CocoaPods | Bước `pod install --repo-update` riêng | Không có | `flutter build ipa` tự chạy pod install; repo chưa có `Podfile` |
| Xcode | `xcode-select` cứng `Xcode_26.4.1`, chạy **sau** bước signing | `maxim-lobanov/setup-xcode` `latest-stable`, chạy **đầu tiên** | Image runner cập nhật sẽ gỡ phiên bản cũ; mọi bước nên dùng cùng một Xcode |
| Build & export IPA | `flutter build ipa` rồi `xcodebuild -exportArchive` riêng | `flutter build ipa --export-options-plist=...` (một lệnh) | Gọn hơn, cùng kết quả |
| ExportOptions | `ios/ExportOptions_<flavor>.plist` trong repo | Sinh trong workflow từ `APPLE_TEAM_ID` + `IOS_BUNDLE_ID` | Không cần file trong repo |
| Bundle ID | Trong project | Ghi đè bằng `update_code_signing_settings(bundle_identifier:)` theo `IOS_BUNDLE_ID` | Workflow và profile match luôn khớp nhau |
| Điều kiện chạy iOS | `platform != 'android'` (payload thiếu `platform` vẫn chạy iOS) | `platform` mặc định `both`, so khớp tường minh | Rõ ràng hơn |
| Script injection | Chèn `${{ github.event.client_payload.* }}` thẳng vào `run:` | Truyền qua `env:` rồi dùng `"$VAR"` | Payload chứa ký tự đặc biệt không thể chạy lệnh tuỳ ý |
| Timeout | Không | `timeout-minutes` 40 (Android) / 45 (iOS) | Build treo không ăn hết quota macOS |
| Concurrency | Không | Cùng tag thì huỷ run cũ | Tránh upload trùng build number |
| Checkout CI repo | `actions/checkout` repo CI | Không | Workflow không cần file nào khác trong repo CI |

### 15.3 Tương thích khi dùng lại công cụ của DGF

- `deploy.sh` hoặc job GitLab của DGF gửi thêm `flavor`, `version`, `build_number`: workflow laboro **bỏ qua** các trường này. Tag vẫn phải đúng định dạng `v<version>+<build>-<dev|prod>`.
- `ENV_FILE_DEV` / `ENV_FILE_PROD` của DGF ở định dạng `.env`; của laboro phải là **JSON**. Không copy giá trị secret giữa hai repo.
- Secret là theo từng repo: keystore, service account, env file, API key của DGF **không dùng được** cho laboro.

### 15.4 Nếu muốn chuyển sang kiểu DGF

Khi laboro có flavor native hoặc logic deploy phức tạp hơn, có thể chuyển logic vào source GitLab:

1. Tạo `ios/Gemfile`, `ios/fastlane/Fastfile` (copy nội dung heredoc trong bước `Write fastlane files`), `ios/ExportOptions.plist` trong repo laboro và commit.
2. Xoá bước `Write fastlane files`; đổi `gem install fastlane` thành `ruby/setup-ruby` với `working-directory: source/ios` + `bundler-cache: true`, và gọi `bundle exec fastlane ...`.
3. Với flavor native: tạo scheme `dev` / `prod` trong Xcode và `productFlavors` trong `build.gradle.kts`, thêm `--flavor $FLAVOR` vào lệnh build, tách ExportOptions theo flavor.
4. Giữ nguyên các cải tiến an toàn: kiểm tra tag, truyền payload qua `env:`, `timeout-minutes`, `concurrency`, credential trong secrets.

