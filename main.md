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

GRUB2から起動できるOSは，特別なものを除き，[Multiboot2](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html)という規格に準拠している必要があります．Multiboot2に対応するためには，Multiboot2 headerと呼ばれるヘッダをバイナリイメージが保持している必要があります．このヘッダはバイナリイメージの先頭から32768バイト以内に存在する必要があり，かつ64ビットのアラインメントを守っている必要があります．詳しくは[仕様書](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html#Specification)を確認してください．

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

GRUB2から直接起動できるOSは32ビットのものに限られます．64ビットOSを起動させる場合，32ビットのブートローダをGRUB2から起動し，ロングモードに遷移したあとにカーネルのコードを実行する必要があります．まずはテスト用に仮のブートローダを作成し，それをGRUB2から起動させます．なお，Multiboot2 headerをバイナリイメージが保持していれば，64ビットOSもGRUB2から起動できるようですが，GRUB2は自動的にロングモードに遷移しないため，いずれにせよロングモードへの遷移するためのコードを書く必要があります．

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
(qemu) info registers
EAX=00000334 EBX=00100090 ECX=00000000 EDX=00000000
ESI=00000000 EDI=00000000 EBP=00000000 ESP=0007ff00
EIP=00000000 EFL=00000046 [---Z-P-] CPL=0 II=0 A20=1 SMM=0 HLT=0
ES =0018 00000000 ffffffff 00cf9300 DPL=0 DS   [-WA]
CS =0010 00000000 ffffffff 00cf9a00 DPL=0 CS32 [-R-]
SS =0018 00000000 ffffffff 00cf9300 DPL=0 DS   [-WA]
DS =0018 00000000 ffffffff 00cf9300 DPL=0 DS   [-WA]
FS =0018 00000000 ffffffff 00cf9300 DPL=0 DS   [-WA]
GS =0018 00000000 ffffffff 00cf9300 DPL=0 DS   [-WA]
LDT=0000 00000000 0000ffff 00008200 DPL=0 LDT
TR =0000 00000000 0000ffff 00008b00 DPL=0 TSS32-busy
GDT=     000010b0 00000020
IDT=     00000000 00000000
CR0=00000011 CR2=00000000 CR3=00000000 CR4=00000000
DR0=0000000000000000 DR1=0000000000000000 DR2=0000000000000000 DR3=0000000000000000
DR6=00000000ffff0ff0 DR7=0000000000000400
EFER=0000000000000000
FCW=037f FSW=0000 [ST=0] FTW=00 MXCSR=00001f80
FPR0=0000000000000000 0000 FPR1=0000000000000000 0000
FPR2=0000000000000000 0000 FPR3=0000000000000000 0000
FPR4=0000000000000000 0000 FPR5=0000000000000000 0000
FPR6=0000000000000000 0000 FPR7=0000000000000000 0000
XMM00=00000000000000000000000000000000 XMM01=00000000000000000000000000000000
XMM02=00000000000000000000000000000000 XMM03=00000000000000000000000000000000
XMM04=00000000000000000000000000000000 XMM05=00000000000000000000000000000000
XMM06=00000000000000000000000000000000 XMM07=00000000000000000000000000000000
```

ブートローダのコード内で`RAX`レジスタに0x334を代入しているため，これは意図通りです．よってブートローダが起動していることが確認できました．

#### リンクをする際の注意点

Multiboot2 headerはバイナリイメージのどこからも参照されていないため，リンクしたあとのバイナリイメージに`.multiboot`セクションが存在しないかもしれません．これは例えば，RustでOSを書いていて，カーネルを一時的に静的ライブラリとしてコンパイルしたあと，リンカを使ってリンクし実際のカーネルを生成する場合に当てはまります．この時，以下の方法でヘッダを強制的にバイナリイメージに付与することが出来ます．

1. ヘッダの前にラベルを用意し，それを`.global`でバイナリ外に公開する．

```asm
    .global start_multiboot_header

start_multiboot_header:
```

2. リンカスクリプトのパラメータとして`--undefined`または短縮形の`-u`をラベルの名前と共に指定する．例えば`--undefined start_multiboot_header`または`-u start_multiboot_header`．リンカスクリプトを使用している場合は，スクリプトに以下の行を追加する．

```ld
EXTERN(start_multiboot_header)
```

参考文献：[How does the -u option for ld work and when is it useful?](https://stackoverflow.com/questions/24834042/how-does-the-u-option-for-ld-work-and-when-is-it-useful)

#### 実際のブートローダの作成

仮のブートローダをQEMU上でGRUB2から起動することが出来たので，いよいよ実際のブートローダを作成していきます．

##### 起動情報の保存

GRUB2がブートローダを起動した直後の`EAX`と`EBX`レジスタの内容は以下のとおりです．他の環境については，[Multiboot2の仕様書の3.3](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html#I386-machine-state)を参照してください．

| レジスタの名前 | 格納されている値                           |
|----------------|--------------------------------------------|
| EAX            | 0x36d76289                                 |
| EBX            | Multiboot2が提供する各情報への物理アドレス |

`EBX`レジスタの内容は勿論のこと，`EAX`レジスタの内容も，OSがMultiboot2を認識しているブートローダによって起動されたことを確認するため，保存する必要があります．保存の方法は様々ですが，ここではカーネルのbssセクションに領域を予め空けておいて，そこに保存するやり方を用います．

カーネルのリンカスクリプトを以下のように編集します．

```ld
    .bss : {
        MULTIBOOT2_SIGNATURE_LMA = . - LMA_OFFSET;
        LONG(0)

        MULTIBOOT2_ADDR_LMA = . - LMA_OFFSET;
        LONG(0)

        /* 後略 */
    }
```

`LMA_OFFSET`は，カーネルの物理アドレスへのオフセットです．例えばカーネルを32ビットでは表せない範囲の仮想アドレスに配置する場合，カーネルが配置される物理アドレスと仮想アドレスは異なります．これらの値を使用して，`EAX`レジスタと`EBX`レジスタの内容を保存します．

```asm
    lea edx, [MULTIBOOT2_SIGNATURE_LMA]
    mov [edx], eax

    lea edx, [MULTIBOOT2_ADDR_LMA]
    mov [edx], eax
```

##### ロングモードへ遷移する

まずはロングモードに遷移します．これにはいくつかの過程が必要です．詳細は[Intelの開発者マニュアル](https://software.intel.com/content/www/us/en/develop/download/intel-64-and-ia-32-architectures-sdm-combined-volumes-1-2a-2b-2c-2d-3a-3b-3c-3d-and-4.html)（以下SDMと呼称）のVolume3の9.8.5に載っています．

1. ページングを無効にする．

[Multiboot2の仕様書](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html#I386-machine-state)によれば，ブートローダによって起動された直後，ページングは無効になっていますが，今後の仕様変更などによって影響されないよう，ここでページングを再度無効にします．これは`CR0`レジスタの最上位ビットを0にすることで可能です．`CR0`レジスタを含む，各コントロールレジスタの各ビットの説明については，SDMのVolume3の2.5に載っています．

```asm
    mov eax, cr0
    and eax, 0x7fffffff
    mov cr0, eax
```

2. Physical-Addres Extensions (PAE) を有効にする

`CR4`レジスタの第5ビットを1にします．

```asm
    mov eax, cr4
    or eax, 0x20
    mov cr4, eax
```

3. ページングの設定を行なう

次にPML4のアドレスを`CR3`レジスタに格納する必要がありますが，まだページテーブルを準備していないため，ここで準備する必要があります．ページングについての説明は省略します．詳しくはSDMのVolume3の4章，または[OSDev Wikiのページングのページ](https://wiki.osdev.org/Paging)を参照してください．

4. ロングモードに遷移する

`IA32_EFER`レジスタの値を変更する必要があります．`IA32_EFER`レジスタの各ビットの意味については，SDMのVolume3のTable 2-1を参照してください．このレジスタを操作するには`rdmsr`命令と`wrmsr`命令を使用する必要があります．`rdmsr`命令と`wrmsr`命令の詳細についてはSDMのVolume2を参照してください．

```asm
mov ecx, 0xc0000080 ; `IA32_EFER`レジスタを指定する．
rdmsr
or eax, 0x100
wrmsr
```

5. ページングを有効にする

これでページングを有効にする準備が出来たので，`CR0`レジスタを操作して，ページングを有効にします．

```asm
mov eax, cr0
or eax, 0x80000000
mov cr0, eax
```

6. スタックポインタを調整する

後にコードセグメントを変更する際に，スタックを使用するため，ここでスタックポインタを変更すると良いです．スタックは，リンカスクリプトを使用してカーネルのバイナリイメージ内に埋め込む方法が楽です．ただし，まだこの段階では32ビットアドレスのみを使用しているため，メモリの下端から4GB以内に用意する必要があります．

例えば以下のように，`.boot`セクション内に，ブートローダ用のスタックを用意します．

```ld
    .boot : {
        *(.boot*)
        . += 1K;
        BOOT_STACK = ALIGN(16);
    }
```

そして，スタックポインタを`BOOT_STACK`が位置するメモリアドレスに変更します．

```asm
    .extern BOOT_STACK

    lea eax, [BOOT_STACK]
    mov esp, eax
```

7. セグメントをロードする

64ビットモード用のセグメントを用意し，これらをロードする必要があります．コードセグメント以外のセグメントには，すべての値が0のセグメント記述子を使用することが出来ます．コードセグメントについてはSDMのVolume3の5.2.1を参照してください．また，`LGDT`命令用の，GDTのアドレスとサイズもメモリ上に用意することを忘れないでください．また，GDTのサイズは，実際のサイズから1を引いたものとなります．GDTは8バイトのアラインメントを守っていると，パフォーマンスが最大になるとされています．詳しくはSDMの3.5.1を参照してください．

以下のコードは，GDTのアドレスとサイズを配置し，次にヌルセグメントを配置したあと，その後にコードセグメントを置いています．

```asm
lgdt_values:
    .word segments_end - segments - 1
    .quad segments

    .align 8
segments:
    // Null
    .long 0
    .long 0
    // Code
    .long 0
    .byte 0
    .byte 0x9a
    .byte 0xa0
    .byte 0
segments_end:
```

セグメントを用意したら，ロードする必要があります．以下のコードは，上記に示したGDTをロードします．

```asm
    lgdt [lgdt_values]
```
