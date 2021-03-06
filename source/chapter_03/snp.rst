细菌基因组SNP差异构建进化树
===========================

1. 需要参考基因组比对的软件
---------------------------

1.1 REALPHY
^^^^^^^^^^^

REALPHY 是一个比较简单易用的软件，它的工作方式是用 Bowtie2 将 fastq 比对到参考基因组获得 SNP 位点，然后用 RaxML 构建 ML 进化树。但是REALPHY 取了一个比较容易起争议的名字，并不是说它创建的进化树就是 “Real” 真正的进化树，这个软件的特点是可以选择不同的参考基因组。

REALPHY 依赖以下软件：

- Bowtie2
- samtools
- RAxML/PhyML
- TREE-PUZZLE
- Phylip

**下载安装**

.. code-block:: bash

   # 安装依赖库
   $ sudo apt install libtool pkg-config zlib1g-dev libncurses5-dev
   # 安装 Java，Ubuntu16.04 目前默认的版本采用 openjdk 1.8
   $ sudo apt install default-jre
   # 下载安装 REALPHY
   $ wget http://realphy.unibas.ch/downloads/REALPHY_v112_exec.zip
   $ unzip REALPHY_v112_exec.zip
   $ sudo mv REALPHY_v112_exec /opt/realphy && sudo chown -R root:root /opt/realphy
   $ sudo touch /opt/realphy/v1.12
   # 创建一个启动脚本，命名为realphy
   $ sudo touch /usr/local/sbin/realphy
   $ sudo chmod +x /usr/local/sbin/realphy

修改 realphy 脚本，内容如下。其中可以根据机器配置和基因组数量调整参数 -Xmx 增大或者减少内存使用数量。

.. code-block:: bash

   #!/bin/bash
   java -Xmx64000m -jar /opt/realphy/RealPhy_v112.jar

**使用Docker容器安装**

.. literalinclude:: realphy.docker
   :language: bash

**构建进化树**

a. 最基本用法，将参考基因组 reference（最好是完成的基因组） 放入 input 文件夹，可以是 fasta 和 genbank 格式，如果放置多个 fasta/gbk 文件，软件会随机选择其中一个作为参考基因组，并将其他序列作为比对序列。如果指定某个序列作为参考基因组，用 `-ref` 参数指定。fastq 数据作为比对数据放入 input 文件夹，如果是PE测序的数据，双端的fastq文件名要用 *_R1.fastq\/\*_R2.fastq 区分。fastq 数据建议先对测序数据进行一定的QC操作，尽可能减少测序错误引入的 SNP 位点。

.. code-block:: bash

   $ realphy input output -ref ref_genome

b. 对核心基因组构建进化树。默认的方式是对参比基因组全部区域比对，包括非编码区。如果希望仅对CDS进行比对，就可以通过添加 `-genes` 参数。如果添加了 -genes 参数，那么参考基因组文件格式必须是含有 CDS 的 genbank 格式数据。

.. code-block:: bash

   $ realphy input output -ref ref_genome -genes

c. 使用多个参考基因组。这是 REALPHY 的一个“卖点”，作者认为当参考基因组与比对基因组差异较大时，如>5%，就会造成进化树构建的偏差。所以通过选择多个参考基因组，平衡不同型别的基因组，减少远源基因组带来的比对差异。通过设置 -refN 来实现多个参考基因组，并用 -merge 参数合并结果。由于计算量较大，不建议使用太多参考基因组。

.. code-block:: bash

   $ realphy input output -ref1 ref1_genome -ref2 ref2_genome -ref3 ref3_genome -merge

建议使用 RAxML 来构建基于 ML 的进化树。因此生成的进化树图为 "PolySeqOut/RAxML_bestTree.raxml"，将其复制到带 GUI 的计算机上，用画树软件生成树图。

.. code-block:: bash

   $ scp user@server-ip:~/data/REF/PolySeqOut/RAxML_bestTree.raxml .
   $ figtree RAxML_bestTree.raxml

--------------------------------------------------------------------------------

1.2 Snippy
^^^^^^^^^^

**下载安装**

.. code-block:: bash

   $ git clone https://github.com/tseemann/snippy
   $ echo 'export PATH="$HOME/repos/snippy/bin:$HOME/repos/snippy/binary/linux:$PATH"' >> ~/.bashrc
   $ source ~/.bashrc

**使用Snippy**

.. code-block:: bash

   # 单个样本用 snippy 分别比对获得每个样本的 snps
   $ snippy --cpus 20 --outdir S01 -ref reference.fa --R1 S01_R1_L001.fastq.gz --R2 S01_R2_L001.fastq.gz
   $ snippy --cpus 20 --outdir S02 -ref reference.fa --R1 S02_R1_L001.fastq.gz --R2 S02_R2_L001.fastq.gz
   ...
   # 将所有 snps 结果汇总，获得 core snps
   $ snippy-core --prefix core-snps S01 S02 S03 ...
   # 单行脚本完成所有样本 snps 比对工作
   $ for i in $(awk -F'_' '{print $1}' <(ls -D *.fastq.gz) | sort | uniq); \
   > do snippy --cpus 20 --outdir $i -ref reference.fa --R1 $i*R1*.fastq.gz --R2 $i*R2*.fastq.gz; \
   > done
   $ snippy-core --prefix core-snps S*
   # 生成 clustal 格式的 core.aln 比对文件，这里用 splittree 构建进化树
   $ splittree -i core-snps/core.aln
   # 或者将生成的 .aln 格式比对文件转换成 .phy 格式，然后再用 raxml 构建 ML 树
   $ raxml -f a -x 12345 -# 100  -p 12345 -m GTRGAMMA -s core.aln -n ex -T 40

2. 不需要参考基因组的软件
-------------------------

2.1 kSNP
^^^^^^^^

kSNP 采用的是基于 kmer 的算法，不需要进行序列的多重比对，也不需要参考基因组进行 Mapping。kSNP 可以分析细菌完成基因组和未完成基因组，也可以直接分析测序 reads 数据。

**下载安装**

.. code-block:: bash

   # 下载软件包，并解压缩到 /opt/ksnp 目录
   $ wget http://nchc.dl.sourceforge.net/project/ksnp/kSNP3.021_Linux_package.zip
   $ sudo unzip kSNP3.021_Linux_package.zip
   $ sudo mv kSNP3.021_Linux_package /opt/ksnp
   $ cd /opt/ksnp
   # 添加版本信息文件
   $ touch v3.021

**使用kSNP3**

.. code-block:: bash

   $
