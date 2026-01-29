# Troubleshooting SPF PermError: Redundant SPF Records

## Masalah

Ketika mengirim email dari domain, terjadi error:

```
spfquery --scope mfrom --id info@preaucess.fr --ip 209.85.128.65 --helo-id mail-wm1-f65.google.com:
permerror
preaucess.fr: Redundant applicable 'v=spf1' sender policies found

Received-SPF: permerror (preaucess.fr: Redundant applicable 'v=spf1' sender policies found)
```

## Penyebab

Domain memiliki **lebih dari satu record TXT SPF** di DNS:

```dns
; Record SPF pertama
preaucess.fr. IN TXT "v=spf1 a mx a:ns337636.ip-5-196-77.eu ip4:46.105.49.131 ip4:5.196.77.200 ip4:91.121.34.12 include:_spf.google.com ~all"

; Record SPF kedua (DUPLIKAT - TIDAK VALID)
preaucess.fr. IN TXT "v=spf1 +a +mx ip4:5.196.77.200 ~all"
```

Menurut **RFC 7208 Section 3.2**, sebuah domain **HANYA BOLEH memiliki SATU record SPF**:

> "A domain name MUST NOT have multiple records that would cause an authorization check to select more than one record."

Ketika ada lebih dari satu record SPF, mail server akan mengembalikan **PermError** dan validasi gagal.

## Dampak

1. **SPF PermError** - Validasi SPF gagal secara permanen
2. **DMARC Failure** - Karena SPF gagal, DMARC juga akan gagal (kecuali DKIM lulus)
3. **Email ditolak atau masuk spam** - Server penerima mungkin menolak atau menandai email sebagai spam

## Solusi

### Langkah 1: Identifikasi record SPF yang ada

```bash
dig TXT preaucess.fr +short
# atau
nslookup -type=TXT preaucess.fr
```

### Langkah 2: Hapus record SPF duplikat

Masuk ke panel DNS provider dan hapus salah satu record SPF, sisakan hanya satu.

### Langkah 3: Gabungkan jika perlu

Jika kedua record memiliki informasi penting, gabungkan menjadi satu record:

**SEBELUM (SALAH - 2 record):**
```
v=spf1 a mx a:ns337636.ip-5-196-77.eu ip4:46.105.49.131 ip4:5.196.77.200 ip4:91.121.34.12 include:_spf.google.com ~all
v=spf1 +a +mx ip4:5.196.77.200 ~all
```

**SESUDAH (BENAR - 1 record):**
```
v=spf1 a mx a:ns337636.ip-5-196-77.eu ip4:46.105.49.131 ip4:5.196.77.200 ip4:91.121.34.12 include:_spf.google.com ~all
```

### Langkah 4: Verifikasi

Setelah perubahan DNS propagasi (bisa memakan waktu 1-48 jam), verifikasi:

```bash
# Harus hanya menampilkan SATU record SPF
dig TXT preaucess.fr +short | grep "v=spf1"

# Test SPF
spfquery --scope mfrom --id info@preaucess.fr --ip 209.85.128.65 --helo-id mail-wm1-f65.google.com
```

## Best Practices untuk SPF

1. **Satu record SPF per domain** - Jangan pernah memiliki lebih dari satu
2. **Batasi DNS lookups** - Maksimal 10 DNS lookups dalam satu record SPF
3. **Gunakan `-all` untuk keamanan ketat** - atau `~all` untuk soft fail
4. **Sertakan semua sumber email** - Termasuk ESP seperti Google, Microsoft, dll.

## Contoh Record SPF yang Benar

```dns
; Untuk domain yang menggunakan Google Workspace + server sendiri
v=spf1 a mx ip4:5.196.77.200 include:_spf.google.com -all
```

## Tools untuk Validasi SPF

- [MXToolbox SPF Lookup](https://mxtoolbox.com/spf.aspx)
- [Kitterman SPF Test](https://www.kitterman.com/spf/validate.html)
- [DMARC Analyzer](https://www.dmarcanalyzer.com/spf/checker/)

## Referensi

- [RFC 7208 - Sender Policy Framework (SPF)](https://datatracker.ietf.org/doc/html/rfc7208)
- [RFC 7489 - DMARC](https://datatracker.ietf.org/doc/html/rfc7489)
