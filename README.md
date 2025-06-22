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

- `typedef unsigned char byte;` → representasi data 0–255
- Tipe boolean (`bool`, `true`, `false`) dan konstanta `NULL`

### include/std\_lib.h

Deklarasi fungsi dasar yang akan diimplementasikan:

- `div`, `mod` → pembagian & modulo tanpa operator `/` dan `%`
- `memcpy`, `strlen`, `strcmp`, `strcpy`, `clear` → manipulasi string & memori

### include/kernel.h

- Deklarasi fungsi yang berasal dari `kernel.asm`:
  ```c
  extern void putInMemory(int segment, int address, char character);
  extern int interrupt(int number, int AX, int BX, int CX, int DX);
  ```
- Fungsi utilitas shell: `printString`, `readString`, `clearScreen`

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

Simulasi interrupt dengan menyimpan nilai AX, BX, CX, DX:

```asm
mov ax,[bp+6]      ; AX
mov bx,[bp+8]      ; BX
mov cx,[bp+10]     ; CX
mov dx,[bp+12]     ; DX
```

Lalu memanggil `int 0x00` melalui label `intr:`

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
            // parseCommand(buf); <-- parsing perintah shell dilakukan di sini
        }
    }
}
```

---

### printString

Menampilkan string ke layar, karakter per karakter menggunakan `int 0x10` AH=0x0E:

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

Membaca input karakter sampai Enter ditekan:

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

Menghapus layar dengan interrupt BIOS `int 0x10`:

```c
void clearScreen() {
    interrupt(0x10, 0x0600, 0, 0x184F, 0);  // scroll seluruh layar
    interrupt(0x10, 0x0200, 0, 0, 0);       // atur kursor ke pojok kiri atas
}
```

---

## std\_lib.c - Fungsi Dasar

- `div(a,b)`: Pengulangan pengurangan
- `mod(a,b)`: Sama seperti div tapi sisanya
- `strlen`: Hitung panjang string sampai null terminator
- `strcmp`: Bandingkan dua string karakter demi karakter
- `strcpy`: Salin isi string dari src ke dst
- `clear`: Set semua byte ke 0
- `memcpy`: Salin blok memori dari src ke dst

---

## Perintah Shell

### a. `echo`

Menampilkan input:

```bash
$> echo hello world
hello world
```

Implementasi langsung menggunakan `printString`

---

### b. `grep`

Mencari substring dalam string input:

```bash
$> echo hello world | grep lo
hello
```

- Cek apakah substring argumen ke-2 ada dalam string input

---

### c. `wc`

Menghitung baris, kata, dan karakter dari input:

```bash
$> echo hello world | wc
Baris: 1, Kata: 2, Karakter: 11
```

- Pisah berdasarkan spasi, enter, dan karakter valid lainnya

---

## Makefile dan Build System

### Build perintah:

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
