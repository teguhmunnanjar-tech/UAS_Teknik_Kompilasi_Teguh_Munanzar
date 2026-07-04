# UAS_Teknik_Kompilasi_Teguh_Munanzar

NAMA : Teguh munanzar aliek
NIM : 231011403062
KELAS : 06TPLE003

# Tugas Proyek Akhir — Representasi Tahapan Kompilasi
### Konstruksi yang Dipilih: **Perulangan (For Loop)**

---

## 1. Pilihan Konstruksi

Konstruksi sintaksis yang diimplementasikan adalah **perulangan `for`**, dengan bentuk umum:

```
for (init; condition; update) { statements }
```

Contoh konkret yang digunakan sebagai kasus uji:

```
for (i = 0; i < 5; i = i + 1) { total = i * 2; }
```

---

## 2. Pattern / Grammar (BNF)

Aturan sintaksis konstruksi `for` didefinisikan menggunakan **Backus-Naur Form (BNF)** sebagai berikut:

```
<for_stmt>   ::= "for" "(" <init> ";" <condition> ";" <update> ")" "{" <block> "}"
<init>       ::= <id> "=" <expr>
<condition>  ::= <id> <relop> <expr>
<update>     ::= <id> "=" <id> <addop> <expr>
<block>      ::= <statement> { <statement> }
<statement>  ::= <id> "=" <expr> ";"
<expr>       ::= <term> { <addop> <term> }
<term>       ::= <id> | <number>
<relop>      ::= "<" | ">" | "<=" | ">=" | "==" | "!="
<addop>      ::= "+" | "-" | "*" | "/"
<id>         ::= letter { letter | digit }
<number>     ::= digit { digit }
```

Grammar ini membatasi bentuk `for` menjadi: satu pernyataan inisialisasi, satu kondisi relasional, satu pernyataan update, dan satu atau lebih pernyataan assignment di dalam badan perulangan (`block`).

---

## 3. Implementasi Program

Program diimplementasikan dalam **Python** (`compiler_for_loop.py`) dan dibagi menjadi 4 tahap utama, sesuai arsitektur kompilator klasik:

```
Source Code → [Lexer] → Tokens → [Parser] → AST → [Semantic Analyzer] → AST tervalidasi → [TAC Generator] → Three-Address Code
```

### 3.1 Analisis Leksikal (Lexer)

Tahap ini memecah source code menjadi rangkaian **token** menggunakan pendekatan **regular expression** (bukan `split()` sederhana seperti pada contoh if-else, melainkan tokenizer berbasis pola agar lebih presisi dan dapat dikembangkan).

Jenis token yang dikenali:

| Jenis Token | Contoh |
|---|---|
| `KEYWORD` | `for` |
| `ID` | `i`, `total` |
| `NUMBER` | `0`, `5`, `1`, `2` |
| `RELOP` | `<`, `>`, `<=`, `>=`, `==`, `!=` |
| `ASSIGN` | `=` |
| `ADDOP` | `+`, `-`, `*`, `/` |
| `LPAREN` / `RPAREN` | `(` `)` |
| `LBRACE` / `RBRACE` | `{` `}` |
| `SEMI` | `;` |

Fungsi `lexical_analysis()` menggunakan satu "master regex" gabungan dari seluruh pola token, lalu mengiterasi setiap kecocokan pada source code dan mengubahnya menjadi objek `Token(type, value)`. Karakter yang tidak dikenali akan memicu `SyntaxError`.

### 3.2 Analisis Sintaksis (Parser → AST)

Parser dibangun dengan metode **recursive descent**, mengikuti aturan BNF di atas. Setiap fungsi parser (`parse_for`, `parse_assignment`, `parse_condition`, `parse_expr`, `parse_term`) berkorespondensi langsung dengan satu aturan grammar.

Hasil parsing berupa **Abstract Syntax Tree (AST)** yang direpresentasikan dengan kelas-kelas berikut:

- `ForStmt(init, condition, update, body)` — node akar perulangan
- `Assign(target, expr)` — pernyataan penugasan
- `Condition(left, op, right)` — ekspresi relasional
- `BinOp(left, op, right)` — operasi biner (`+ - * /`)
- `Var(name)` dan `Num(value)` — node daun (leaf)

Contoh AST yang dihasilkan dari `for (i = 0; i < 5; i = i + 1) { total = i * 2; }`:

```
ForStmt(
  init      = Assign(Var(i) = Num(0)),
  condition = Condition(Var(i) < Num(5)),
  update    = Assign(Var(i) = BinOp(Var(i) + Num(1))),
  body      = [Assign(Var(total) = BinOp(Var(i) * Num(2)))]
)
```

### 3.3 Analisis Semantik

Kelas `SemanticAnalyzer` melakukan dua jenis pengecekan dasar menggunakan **tabel simbol** (`symbol_table`):

1. **Validasi keberadaan variabel** — setiap variabel yang *dibaca* (pada kondisi, update, atau ruas kanan assignment di dalam body) harus sudah dideklarasikan sebelumnya (melalui `init` atau assignment lain). Jika tidak, program memunculkan `SemanticError`.
2. **Validasi tipe data** — seluruh nilai dalam konstruksi ini bertipe `int`. Analyzer memeriksa bahwa kedua operand dari sebuah operasi biner memiliki tipe yang sama, dan juga mendeteksi potensi **pembagian dengan nol** ketika pembagi berupa literal `0`.

Program utama (`compile_for_loop`) mendemonstrasikan dua skenario:

- **Skenario valid**: `for (i = 0; i < 5; i = i + 1) { total = i * 2; }` → lolos analisis semantik, menghasilkan tabel simbol `{'i': 'int', 'total': 'int'}`.
- **Skenario gagal**: `for (i = 0; y < 5; i = i + 1) { total = i * 2; }` → variabel `y` belum pernah dideklarasikan, sehingga analyzer melempar error:
  `Variabel 'y' digunakan sebelum dideklarasikan.`

### 3.4 Generasi Kode Antara (Three-Address Code)

Kelas `TACGenerator` menerjemahkan AST yang telah tervalidasi menjadi **Three-Address Code**, dengan aturan penerjemahan `for` sebagai berikut:

```
      <kode init>
L1:
      ifFalse (<kondisi>) goto L2
      <kode body>
      <kode update>
      goto L1
L2:
```

Setiap ekspresi biner yang kompleks (misalnya `i * 2` atau `i + 1`) diuraikan menjadi variabel sementara (`t1`, `t2`, dst.) melalui fungsi rekursif `gen_expr()`, sehingga setiap instruksi TAC hanya memuat maksimal satu operator — sesuai prinsip *three-address code*.

**Hasil TAC** untuk `for (i = 0; i < 5; i = i + 1) { total = i * 2; }`:

```
i = 0
L1:
ifFalse (i < 5) goto L2
t1 = i * 2
total = t1
t2 = i + 1
i = t2
goto L1
L2:
```

---

## 4. Cara Menjalankan Program

```bash
python3 compiler_for_loop.py
```

Program akan menampilkan output berurutan sesuai keempat tahap kompilasi (leksikal → sintaksis → semantik → TAC) untuk kasus valid, kemudian menampilkan contoh penanganan kesalahan semantik pada kasus kedua.

---

## 5. Kesimpulan

Implementasi ini menunjukkan bagaimana satu konstruksi bahasa pemrograman (`for`) diproses melalui empat tahap utama kompilator:

1. **Leksikal** mengubah teks mentah menjadi token-token bermakna.
2. **Sintaksis** menyusun token menjadi struktur pohon (AST) sesuai grammar BNF.
3. **Semantik** memvalidasi kebenaran logis program (deklarasi variabel & kesesuaian tipe) di atas AST.
4. **Generasi Kode** menerjemahkan AST tervalidasi menjadi Three-Address Code yang merepresentasikan alur kontrol perulangan menggunakan label dan lompatan (`goto`), siap untuk tahap optimasi atau generasi kode mesin selanjutnya.
