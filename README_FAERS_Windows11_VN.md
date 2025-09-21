# PharmApp FAERS — Hướng dẫn cài đặt & khôi phục trên **Windows 11** (Tiếng Việt)

Tài liệu này hướng dẫn bạn thiết lập **conda environment**, khởi tạo **PostgreSQL**, và **khôi phục dữ liệu FAERS** từ file dump trên **Windows 11**.

> Đã kiểm thử với PostgreSQL **17.x** và Python **3.11**.  
> Khuyến nghị đặt tên môi trường conda là **`pharmapp_3`** (bạn cũng có thể dùng lại env cũ `py311_rdkit_psql` nếu đã có).

---

## 0) Kho dữ liệu & Tham khảo

- Repo FAERS: **https://github.com/nghiencuuthuoc/pharmapp-faers**  
- Repo tham khảo ChEMBL: **https://github.com/nghiencuuthuoc/pharmapp-chembl-35**  
- Dump FAERS (Google Drive): **https://drive.google.com/drive/folders/1x8U87NpS3uRuxpHBRsxZa7LvhdlGeTCC?usp=sharing**

> Gợi ý đặt file dump tại: `E:\PharmAppDev\database\FAERS\faers_db_YYYYMMDD.dump`

---

## 1) Tạo & kích hoạt **conda env**

Mở **Anaconda Prompt** (khuyến nghị) hoặc CMD, sau đó chạy:

```cmd
conda create -n pharmapp_3 python=3.11 -y
conda activate pharmapp_3

REM Thư viện lõi & PostgreSQL client/server từ conda
conda install -c conda-forge rdkit rdkit-postgresql matplotlib vs2015_runtime postgresql -y

REM Bộ build & image (phục vụ các module PharmApp)
conda install -c conda-forge -c anaconda cmake cairo pillow eigen pkg-config -y
conda install -c conda-forge -c anaconda boost-cpp boost py-boost -y
```

Kiểm tra nhanh công cụ có sẵn trên PATH:
```cmd
psql --version
pg_restore --version
```

---

## 2) Khởi tạo **PostgreSQL cluster** cục bộ

Chọn thư mục data, ví dụ: `E:\PharmAppDev\pgdata`

```cmd
initdb -D "E:\PharmAppDev\pgdata"
pg_ctl -D "E:\PharmAppDev\pgdata" -l logfile start
```

Lệnh hữu ích khác:
```cmd
pg_ctl -D "E:\PharmAppDev\pgdata" status
pg_ctl -D "E:\PharmAppDev\pgdata" restart
pg_ctl -D "E:\PharmAppDev\pgdata" stop -m fast
```

> **Lưu ý** khi viết vào file `.bat` cần escape dấu ngoặc kép bằng `^` (nếu dùng):  
> `pg_ctl -D ^"E^:^\PharmAppDev^\pgdata^" -l logfile start`

Nếu bạn đã cài PostgreSQL bằng installer chính thức và chạy như **Windows Service**, hãy dùng service đó hoặc chỉnh port/data directory cho phù hợp.

---

## 3) Tạo **role** & **database** cho FAERS

Chạy các lệnh sau (sẽ hỏi mật khẩu nếu user `postgres` có đặt):

```cmd
psql -h localhost -U postgres -c "DO $$ BEGIN IF NOT EXISTS (SELECT 1 FROM pg_roles WHERE rolname='nct') THEN CREATE ROLE nct LOGIN PASSWORD 'nct'; ALTER ROLE nct CREATEDB; END IF; END $$;"
psql -h localhost -U postgres -c "DROP DATABASE IF EXISTS faers_db;"
psql -h localhost -U postgres -c "CREATE DATABASE faers_db OWNER nct TEMPLATE template0 ENCODING 'UTF8';"
```

> Nếu gặp lỗi về `LC_COLLATE/LC_CTYPE`, hãy dùng lệnh tạo DB như trên (không ép LC), hoặc chọn LC hợp lệ trên hệ thống bạn.

---

## 4) Tải **dump** & **khôi phục dữ liệu**

1. Tải file dump từ Google Drive ở trên.
2. Đặt file vào: `E:\PharmAppDev\database\FAERS\faers_db_YYYYMMDD.dump` (ví dụ: `faers_db_20250921.dump`).

Khôi phục (song song) bằng `pg_restore`:
```cmd
set PGPASSWORD=nct
pg_restore -h localhost -U nct -d faers_db ^
  --clean --if-exists --no-owner --no-privileges --role=nct ^
  -j 6 "E:\PharmAppDev\database\FAERS\faers_db_20250921.dump"
set PGPASSWORD=
```

- `-j 6`: dùng 6 luồng song song (tùy CPU/SSD có thể 4–12).
- `--no-owner --role=nct`: object sau restore thuộc user `nct`.
- `--clean --if-exists`: xóa object cũ trước khi tạo lại.

---

## 5) Kiểm tra sau khi restore

```cmd
psql -h localhost -U nct -d faers_db -c "\dt+"
psql -h localhost -U nct -d faers_db -c "SELECT current_user, current_database();"
```

Thống kê nhanh kích thước & ước lượng số dòng:
```cmd
psql -h localhost -U nct -d faers_db -c "SELECT relname AS table_name, pg_size_pretty(pg_total_relation_size(relid)) AS total_size, n_live_tup AS approx_rows FROM pg_stat_user_tables ORDER BY total_size DESC LIMIT 20;"
```

Tối ưu plan (cập nhật statistics):
```cmd
psql -h localhost -U nct -d faers_db -c "VACUUM (ANALYZE);"
```

---

## 6) Tạo **index** khuyến nghị (chạy 1 lần)

```cmd
psql -h localhost -U nct -d faers_db -c "CREATE INDEX IF NOT EXISTS idx_drug_primaryid ON drug(primaryid);"
psql -h localhost -U nct -d faers_db -c "CREATE INDEX IF NOT EXISTS idx_reac_primaryid ON reac(primaryid);"
psql -h localhost -U nct -d faers_db -c "CREATE INDEX IF NOT EXISTS idx_ther_primaryid ON ther(primaryid);"
psql -h localhost -U nct -d faers_db -c "CREATE INDEX IF NOT EXISTS idx_drug_drugname_lower ON drug (LOWER(drugname));"
psql -h localhost -U nct -d faers_db -c "CREATE INDEX IF NOT EXISTS idx_reac_pt_lower ON reac (LOWER(pt));"
```

---

## 7) Truy vấn mẫu (sanity check)

```cmd
psql -h localhost -U nct -d faers_db -c "SELECT UPPER(drugname) AS drug, COUNT(*) AS n FROM drug WHERE role_cod IN ('PS','SS') GROUP BY 1 ORDER BY n DESC LIMIT 20;"
psql -h localhost -U nct -d faers_db -c "SELECT pt, COUNT(*) AS n FROM reac GROUP BY pt ORDER BY n DESC LIMIT 20;"
psql -h localhost -U nct -d faers_db -c "SELECT UPPER(d.drugname) AS drug, r.pt, COUNT(*) AS n FROM drug d JOIN reac r ON r.primaryid = d.primaryid WHERE d.role_cod IN ('PS','SS') GROUP BY 1,2 ORDER BY n DESC LIMIT 20;"
psql -h localhost -U nct -d faers_db -c "SELECT SUBSTRING(event_dt,1,4) AS year, COUNT(*) AS n FROM demo GROUP BY 1 ORDER BY 1;"
```

---

## 8) Lỗi thường gặp & cách xử lý

- **Thiếu extension** (ví dụ `pgcrypto`, `"uuid-ossp"`):
  ```cmd
  psql -h localhost -U postgres -d faers_db -c "CREATE EXTENSION IF NOT EXISTS pgcrypto;"
  psql -h localhost -U postgres -d faers_db -c "CREATE EXTENSION IF NOT EXISTS \"uuid-ossp\";"
  ```

- **Không tìm thấy `psql/pg_restore/pg_ctl`**:
  - Mở **Anaconda Prompt** (đã `conda activate pharmapp_3`), hoặc
  - Đảm bảo PostgreSQL (từ conda hoặc installer) đã thêm vào `PATH`.

- **Lỗi quyền/owner**:
  - Duy trì `--no-owner --role=nct` trong `pg_restore`.
  - Tạo DB với `OWNER nct` như ở bước 3.

- **Khác cổng/đang có instance khác**:
  - Thêm `-p <port>` cho `psql/pg_restore`, hoặc
  - Dừng instance cũ: `pg_ctl -D "E:\PharmAppDev\pgdata" stop -m fast`

- **Tăng tốc**:
  - Tăng `-j` khi `pg_restore` nếu CPU/SSD mạnh.
  - Đặt data trên SSD nhanh.
  - (Nâng cao) Tạm chỉnh `maintenance_work_mem`, `max_wal_size`, `checkpoint_timeout` trước restore rồi phục hồi cấu hình sau.

---

## 9) Gợi ý quy ước & đóng góp

- Dùng tên env thống nhất `pharmapp_3` giữa các module của PharmApp.
- Góp ý/issue/PR tại repo **pharmapp-faers**.
- Ưu tiên script **cross-platform** (Windows/Ubuntu) khi có thể.

---

**Chúc bạn cài đặt & phân tích dữ liệu thuận lợi!**  
Gặp lỗi, vui lòng chụp/ghi đầy đủ log và mở issue trên repo để được hỗ trợ.
