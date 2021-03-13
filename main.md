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

### やり方

#### Multiboot2のヘッダを作成する

GRUB2から起動できるOSは，特別なものを除き，[Multiboot2](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html)という規格に準拠している必要があります．Multiboot2に対応するためには，Multiboot2 headerと呼ばれるヘッダをバイナリイメージが保持している必要があります．このヘッダはバイナリイメージの先頭から32768バイト以内に存在する必要があり，かつ64ビットのアラインメントを守っている必要があります．詳しくは[仕様書](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html#Specification)を確認してください．また，Multiboot2は32ビットの実行ファイルのみが対応していて，64ビットのOSを使用する場合は後述するように32ビットのブートローダを作成し，それを用いてOSを起動させる必要があります．

以下のようなアセンブリファイルを作成します．

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
    .set TERMINATION_SIZE, 8

    .word TERMINATION
    .word TERMINATION_SIZE

end_multiboot_header:
```

Multiboot2 headerの先頭はマジックナンバです．次にアーキテクチャ番号が来ます．アーキテクチャ番号の対応は以下の表のとおりです．

| アーキテクチャ | 番号 |
|----------------|------|
| i386           |    0 |
| MIPS           |    4 |

その次はヘッダサイズです．これはMultiboot2 headerが占めるバイト数です．そしてチェックサムです．これはマジックナンバ，アーキテクチャ番号，そしてヘッダサイズの和のマイナス値です．

#### テスト用のブートローダを作成する

GRUB2から直接起動できるOSは32ビットのものに限られます．64ビットOSを起動させる場合，32ビットのブートローダをGRUB2から起動し，ロングモードに遷移したあとにカーネルのコードを実行する必要があります．まずはテスト用に仮のブートローダを作成し，それをGRUB2から起動させます．

```asm
    .intel_syntax noprefix
    .global _start

    mov eax, 0x334

_start:
    jmp _start
```

このコードは`EAX`レジスタに`0x334`を代入したあと無限ループをします．レジスタの値はQEMUのデバッグモニタから確認することが可能なので，これを用いて起動できているかを確認します．

