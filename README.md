# OpenACC_CG_sample
GPU移植のための実装例（共役勾配法によるポアソン方程式求解）

## 概要
* 共役勾配法のC及びFORTRANのコードを、OpenACCでGPU対応する実例です。
* OpenACCでは、元のCPU専用コードに指示文を挿入することでGPU対応するため、下記の場合に特に有用です。
  * 大量の既存コードを最小限の手間でGPU化したい場合
  * 同一コードでCPU機・NVIDIA社製GPU搭載機の両方をサポートしたい場合
* 本レポジトリのプログラムのベースは、[東京大学情報基盤センターお試しアカウント付き並列プログラミング講習会「並列有限要素法で学ぶ並列プログラミング徹底入門」](http://nkl.cc.u-tokyo.ac.jp/FEMall/)で解説されている、CPU用のプログラムです。共役勾配法のアルゴリズムや、MPI・OpenMPによる並列化の方法に関しては、上記URLをご覧ください。

## 実行までの流れ

本サンプルコードは直方体格子上でポアソン方程式を解くプログラムです。直方体格子の定義ファイルを事前に生成しておく必要があるため、実行は下記の流れで行います。
1. 主プログラムと格子生成プログラムをコンパイルする
1. 格子生成プログラムを実行する
1. 主プログラムを実行し、ポアソン方程式を解く

ここでは設定ファイルを変更せず、$`128^3`$個の格子を1GPUで処理する手順を示しています。[格子サイズや使用GPU数を変える](#格子サイズや使用GPU数を変える)こともできます。

### 主プログラムと格子生成プログラムのコンパイル
Wisteria Aqariusで実行する場合は、下記コマンドでNVIDIA HPC SDKを利用可能な状態にします。
```
module load nvidia nvmpi cuda
```
次に、ディレクトリ`pmesh/`に移動し、`mpif90 pmesh.f`を実行します。
さらに、ディレクトリ`../src/`に移動し、`make`を実行します。
ファイル`run/prog_gpu`が生成されていれば成功です。

### 格子生成プログラムを実行する
```
cd pmesh/
nvfortran pmesh.f

# Wisteria Aquariusで実行する場合
pjsub pjsub_pmesh.sh
# ワークステーションなどで実行する場合
mpirun -n 1 ./a.out
```
### 主プログラムを実行する
```
cd run/

# Wisteria Aquariusで実行する場合
pjsub pjsub_run.sh
# ワークステーションなどで実行する場合
mpirun -n 1 ./prog_gpu
```
上記の流れを一度行った後にコードや設定を変えずに再実行する場合は、コンパイルと格子生成を省略して主プログラムをすぐに実行できます。

## FORTRAN版の実行速度について
NVIDIA HPC SDK 24.01の時点では、FORTRAN版はC版に比べて1.8倍の時間がかかります。これは、?.Fの?行目の変数?の型を`integer(4)`から`integer(8)`に変えると解消しますが、具体的な原因は不明です。

## 行列生成部分をGPU化する
(ここではFORTRAN版に関して説明していますが、C版も同様です)

サブルーチン?の行列生成処理は、変数?の同一インデックスにループから複数回アクセスする可能性があるため、そのまま並列化するとアクセス競合が起きて誤った結果になるおそれがあります。そのため、標準コードではGPU化せず、CPU上で処理しているため時間がかかります。しかし、`!$acc atomic`を使った排他アクセスを利用すると、競合を避けてGPUで並列化することができます。そのコードは?.F (C版では?.c)に記載しています。Makefileの?.F (C版: ?.c)を?.F (C版: ?.c)に置き換え、`make clean`したうえで`make`すると行列生成GPU版を利用することができます。

## 格子サイズや使用GPU数を変える
1. 格子サイズと使用GPU数を決める
1. 主プログラムと格子生成プログラムの設定ファイルを編集する
1. 格子生成プログラムを再実行する
1. 主プログラムを実行する

### 直方体格子のサイズと使用GPU数を決める
格子数と使用GPU数(並列数)は下記のフォーマットで指定します。
```
(x方向格子数) (y方向格子数) (z方向格子数)
(x方向並列数) (y方向並列数) (z方向並列数)
```
合計で、(x方向並列数)×(y方向並列数)×(z方向並列数)個のGPUを使用します。また、各軸の格子数はその軸の並列数で割り切れる必要があります。

例えば、
```
256 256 256
2 2 2
```
とすると、Aquariusで1ノードに搭載された8個のGPUを全て使って$`256^3`$格子を扱うことができます。

また、6GPUのマシンを使う場合は、
```
240 256 256
3 2 1
```
のように並列数を割り振ることができます。ここでは、x軸を3並列に分割しているため、x方向の格子数を3の倍数としています。

問題サイズを固定し、並列数を変えて速度を比較するstrong scalingの性能を測定する場合は、並列数に様々な値を設定することになるので、そのいずれでも割り切れるように格子数を指定しておく必要があります。