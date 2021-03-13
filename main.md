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
