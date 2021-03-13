### はじめに
今までは自作のブートローダを用いてOSを起動していました．今回はGRUB2を用いたOSの起動に挑戦することにしました．

### 環境

```zsh
gcc (Gentoo 10.2.0-r5 p6) 10.2.0
Copyright (C) 2020 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

```zsh
QEMU emulator version 5.2.0
Copyright (c) 2003-2020 Fabrice Bellard and the QEMU Project developers
```

```zsh
grub-mkrescue (GRUB) 2.05_alpha20200310
```

### やり方

#### Multiboot2のヘッダを作成する

GRUB2から起動できるOSは，特別なものを除き，[Multiboot2](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html)という規格に準拠している必要があります．Multiboot2に対応するためには，Multiboot2 headerと呼ばれるヘッダをバイナリイメージが保持している必要があります．このヘッダはバイナリイメージの先頭から32768バイト以内に存在する必要があり，かつ64ビットのアラインメントを守っている必要があります．詳しくは[仕様書](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html#Specification)を確認してください．また，Multiboot2は32ビットの実行ファイルのみが対応していて，64ビットのOSを使用する場合は後述するように32ビットのブートローダを作成し，それを用いてOSを起動させる必要があります．

以下のようなアセンブリファイルmultiboot.sを作成します．

```asm
    .section .multiboot
    .intel_syntax noprefix

    .extern _start

    .align 64

start_multiboot_header:

    .set MAGIC_NUMBER, 0xe85250d6
    .set ARCHITECTURE_I386, 0
    .set HEADER_SIZE, end_multiboot_header - start_multiboot_header
    .set CHECKSUM, -(MAGIC_NUMBER + ARCHITECTURE_I386 + HEADER_SIZE)

    .long MAGIC_NUMBER
    .long ARCHITECTURE_I386
    .long HEADER_SIZE
    .long CHECKSUM

    .set TERMINATION, 0
    .set TERMINATION_FLAGS, 0
    .set TERMINATION_SIZE, 8

    .word TERMINATION
    .word TERMINATION_FLAGS
    .long TERMINATION_SIZE

end_multiboot_header:
```

Multiboot2 headerの各フィールドは以下のとおりです．

| フィールドの意味   | バイト数   | 値                                                    |
|--------------------|------------|-------------------------------------------------------|
| マジックナンバ     |          4 |                                            0xe85250d6 |
| アーキテクチャ番号 |          4 | 下記参照                                              |
| ヘッダサイズ       |          4 | ヘッダのバイト数                                      |
| チェックサム       |          4 | -(マジックナンバ + アーキテクチャ番号 + ヘッダサイズ) |
| タグ               | 種類による | タグによる                                            |

アーキテクチャ番号の対応は以下の表のとおりです．

| アーキテクチャ | 番号 |
|----------------|------|
| i386           |    0 |
| MIPS           |    4 |

タグは任意数配置することが出来ます．タグの構造はだいたい以下のとおりです．詳しくは[仕様書](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html#Header-tags)を確認してください．

| タグの名前 | バイト数 | 説明                                                        |
|------------|----------|-------------------------------------------------------------|
| type       |        2 | タグの番号                                                  |
| flags      |        2 | もしビット0がセットされているならば，optionalとして扱われる |
| size       |        4 | タグ全体のサイズ                                            |

C言語の文字列における\0のような，タグの終端を表すためのタグが存在します．このタグはtype = 0, size = 8です．

#### テスト用のブートローダを作成する

前述したとおり，Multiboot2が対応しているアーキテクチャは32ビットのみです．したがって，GRUB2から直接起動できるOSは32ビットのものに限られます．64ビットOSを起動させる場合，32ビットのブートローダをGRUB2から起動し，ロングモードに遷移したあとにカーネルのコードを実行する必要があります．まずはテスト用に仮のブートローダを作成し，それをGRUB2から起動させます．

以下のような仮のブートローダmain.sを作成します．

```asm
    .intel_syntax noprefix
    .global _start

    mov eax, 0x334

_start:
    jmp _start
```

このコードは`EAX`レジスタに`0x334`を代入したあと無限ループをします．レジスタの値はQEMUのデバッグモニタから確認することが可能なので，これを用いて起動できているかを確認します．

#### ブートローダ用のリンカスクリプトを作成する

Multiboot2 headerをバイナリイメージの先頭に位置させるために，リンカスクリプトを用いてヘッダの位置を指定します．

以下のようなリンカスクリプトbootloader.ldを作成します．

```ld
OUTPUT_FORMAT(elf32-i386)

ENTRY(_start)

SECTIONS
{
    .multiboot ALIGN(8) : {
        *(.multiboot*)
    }

    .text : {
        *(.text*)
    }

    .data : {
        *(.data*)
    }

    .rodata : {
        *(.rodata*)
    }

    .bss : {
        *(.bss*)
    }

    .eh_frame : {
        *(.eh_frame)
    }
}
```

各セクションは記述した順番と同じようにバイナリイメージ上に並びます．また，`ALIGN`を使用して64ビットのアラインメント制約を守ります．

#### ブートローダイメージを作成する

以下のコマンドを使用してブートローダイメージを作成します．

```zsh
gcc -o bootloader -T bootloader.ld multiboot.s main.s
```

これで`bootloader`と名付けられたブートローダイメージが生成されます．

#### GRUB2の設定ファイルを作成する

GRUB2の設定ファイルを作成し，OSを起動するためのエントリを作成します．

以下のファイルをgrub.cfgという名前で保存します．

```conf
menuentry "OS"{
    multiboot2 /boot/bootloader
    boot
}
```

これで，"OS"という項目が定義されます．

#### GRUB2のイメージを作成する

QEMU上でテストするために，GRUB2のイメージを作成します．これは[GRUBで簡単なOSカーネルを動かしてみる](http://inaz2.hatenablog.com/entry/2015/12/31/221319)というページを参考にしました．

```zsh
mkdir -p isofiles/boot/grub
cp bootloader isofiles/boot/
cp grub.cfg isofiles/boot/grub/
```

以下のようなディレクトリ構成になっているはずです．

```
> tree isofiles
isofiles
└── boot
    ├── bootloader
    └── grub
        └── grub.cfg

2 directories, 2 files
```

ファイルを準備したら，以下のコマンドを使用してイメージを作成します．

```zsh
grub-mkrescue -o os.iso isofiles
```

これで，`os.iso`と名付けられたGRUB2のイメージファイルが生成されました．

#### QEMUでテストする

以下のコマンドを使用してQEMUを起動します．

```zsh
qemu-system-x86_64 -drive format=raw,file=os.iso --monitor stdio
```

QEMUが起動し，GRUB2のメニュー画面が表示されます．エントリを選択しエンターキを押しても何も面白いことは起こりません．

しかし，QEMUを起動したターミナルで以下のコマンドを入力すると，各レジスタの値が表示され，`RAX`レジスタの値が(0x)334となっているはずです．

```
info registers
```

ブートローダのコード内で`RAX`レジスタに0x334を代入しているため，これは意図通りです．よってブートローダが起動していることが確認できました．
