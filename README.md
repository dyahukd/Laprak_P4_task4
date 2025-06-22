# Laprak_P4_task4

## Deskripsi Umum

Pada praktikum ini, kita diminta membuat sebuah sistem operasi mini bernama **LilHabOS** yang berjalan di atas emulator Bochs. Sistem operasi ini mendukung shell sederhana dan beberapa fitur seperti `echo`, `grep`, dan `wc` — termasuk kemampuan **piping** antar perintah layaknya di Unix/Linux.

## Struktur Direktori Proyek

```
task-3/
├── bin/                # File hasil kompilasi dan floppy.img
├── include/            # Header file
│   ├── kernel.h
│   ├── std_lib.h
│   └── std_type.h
├── src/                # Kode sumber utama
│   ├── bootloader.asm
│   ├── kernel.asm
│   ├── kernel.c
│   └── std_lib.c
├── bochsrc.txt         # Konfigurasi emulator Bochs
└── makefile            # Konfigurasi kompilasi otomatis
```

---

## Penjelasan Header File

### include/std\_type.h

Header ini dikhususkan untuk mendefinisikan tipe data standar yang dipakai sepanjang proyek LilHabOS. Isi filenya telah disederhanakan namun tetap fungsional:

- `typedef char bool;` → Tipe boolean didefinisikan sebagai `char`
- Konstanta:
  - `NULL` → Di-set sebagai `0`
  - `true` → Nilai boolean `1`
  - `false` → Nilai boolean `0`

Revisi ini menjadikan kode lebih ringan dan tetap readable.

---

### include/kernel.h

Header ini berisi deklarasi fungsi-fungsi kernel, baik yang ditulis di Assembly (`kernel.asm`) maupun di C (`kernel.c`). Setelah direvisi, fungsi-fungsinya mencakup:

```c
void printString(char* str);
void readString(char* buf);
void clearScreen();
void runCommand(char* input);
void executePipe(char* cmd1, char* cmd2);
void handleCommand(char* cmd, char* input);
void putInMemory(int segment, int address, char c);
int interrupt(int number, int AX, int BX, int CX, int DX);
void executePipeline(char* cmds[], int n);
```

Beberapa fungsi baru yang ditambahkan dalam revisi adalah:

- `runCommand` → Mengeksekusi perintah shell
- `executePipe` → Menangani pipe antar dua perintah (misalnya `echo ... | wc`)
- `handleCommand` → Menentukan handler yang tepat untuk masing-masing command
- `executePipeline` → Versi lanjutan dari piping, mendukung chaining lebih dari dua command

Penambahan fungsi-fungsi ini bikin sistem lebih modular dan mudah dikembangkan.

---

## Penjelasan Kernel.asm

### 1. `putInMemory`

Menulis karakter ke alamat memori:

```asm
mov ax,[bp+4]      ; segment
mov si,[bp+6]      ; address
mov cl,[bp+8]      ; character
mov ds,ax
mov [si],cl
```

### 2. `interrupt`

Simulasi pemanggilan interrupt, menyimpan nilai AX, BX, CX, DX:

```asm
mov ax,[bp+6]      ; AX
mov bx,[bp+8]      ; BX
mov cx,[bp+10]     ; CX
mov dx,[bp+12]     ; DX
```

Kemudian dipanggil dengan `int 0x00` melalui label `intr:`.

---

## Implementasi `kernel.c`

### Fungsi utama:

```c
int main() {
    char buf[128];
    clearScreen();
    printString("LilHabOS - A01\n");

    while (true) {
        printString("$> ");
        readString(buf);
        printString("\n");

        if (strlen(buf) > 0) {
            runCommand(buf);
        }
    }
}
```

Yang sebelumnya `parseCommand(buf);`, kini diganti menjadi `runCommand(buf);` sesuai revisi yang ada pada `kernel.h`. Fungsi `runCommand` menangani parsing dan eksekusi perintah termasuk pipe.

---

### printString

Fungsinya tetap sama, menampilkan string ke layar:

```c
void printString(char* str) {
    int i = 0;
    while (str[i] != '\0') {
        interrupt(0x10, 0x0E00 + str[i], 0, 0, 0);
        i++;
    }
}
```

### readString

Baca input keyboard hingga Enter ditekan. Juga menangani Backspace:

```c
void readString(char* buf) {
    int i = 0;
    char c;
    do {
        c = interrupt(0x16, 0, 0, 0, 0);
        if (c == 0x08 && i > 0) {
            printString("\b \b");
            i--;
        } else if (c != 0x08) {
            buf[i++] = c;
            interrupt(0x10, 0x0E00 + c, 0, 0, 0);
        }
    } while (c != '\r');
    buf[i] = '\0';
}
```

### clearScreen

Membersihkan layar dengan BIOS interrupt:

```c
void clearScreen() {
    interrupt(0x10, 0x0600, 0, 0x184F, 0);
    interrupt(0x10, 0x0200, 0, 0, 0);
}
```

---

## std\_lib.c - Fungsi Dasar

Fungsi-fungsi dasar manipulasi string/memori masih sama dan berfungsi sebagai utilitas:

- `div`, `mod` → Operasi pembagian dan modulo tanpa operator bawaan
- `strlen`, `strcmp`, `strcpy` → Operasi dasar string
- `clear`, `memcpy` → Bersihkan dan salin blok memori

---

## Perintah Shell

### a. `echo`

Menampilkan kembali input user:

```bash
$> echo hello world
hello world
```

Implementasi cukup langsung menggunakan `printString`.

---

### b. `grep`

Menampilkan baris yang mengandung substring tertentu:

```bash
$> echo hello world | grep lo
hello
```

Dalam implementasinya, program akan mengecek apakah substring dari argumen kedua muncul di input pertama.

---

### c. `wc`

Menghitung jumlah baris, kata, dan karakter:

```bash
$> echo hello world | wc
Baris: 1, Kata: 2, Karakter: 11
```

Menggunakan pemisah seperti spasi dan newline untuk menghitung kata dan baris.

---

## Makefile dan Build System

Isi Makefile tidak banyak berubah, hanya saja diperbaiki dan disusun lebih rapi. Berikut proses build-nya:

```makefile
build: prepare bootloader stdlib kernel link

prepare:
	dd if=/dev/zero of=bin/floppy.img bs=512 count=2880

bootloader:
	nasm -f bin src/bootloader.asm -o bin/bootloader.bin

stdlib:
	bcc -ansi -c -o bin/std_lib.o src/std_lib.c

kernel:
	bcc -ansi -c -o bin/kernel.o src/kernel.c
	nasm -f as86 src/kernel.asm -o bin/kernel_asm.o

link:
	ld86 -o bin/kernel.bin -d bin/kernel.o bin/kernel_asm.o bin/std_lib.o
	cat bin/bootloader.bin bin/kernel.bin > bin/floppy.img

run:
	bochs -f bochsrc.txt
```

---

## Testing & Output

1. Jalankan build:

```bash
make build
```

2. Jalankan emulator Bochs:

```bash
make run
```

3. Tes perintah:

- `$> echo halo`
- `$> echo halo dunia | grep lo`
- `$> echo halo dunia | wc`

---

## Screenshot Hasil

![Image](https://github.com/user-attachments/assets/4db1a804-ffd2-4c34-8c0d-4fb92916a250)
