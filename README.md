# DTI

We implement our method based on the codebase of [fairseq](https://github.com/pytorch/fairseq). 

# Requirements and Installation
* PyTorch version == 1.10.2
* Fairseq version == 0.10.2
* RDKit version == 2020.09.5

To install the code from source
```shell
# Need to modify url when public
git clone git@github.com:QizhiPei/DTI.git
cd DTI
pwd=$PWD

git clone git@github.com:pytorch/fairseq.git /tmp/fairseq
cd /tmp/fairseq
git checkout v0.10.2

cd $pwd
cp -r -n /tmp/fairseq/* ./

conda create -n fairseq-dti python=3.7
conda activate fairseq-dti
conda install -c conda-forge rdkit
pip install future scipy sklearn
pip install -e . 
```
# Getting Started
## Joint Training

### Data Preprocessing

#### Unlabeled Molecule and Protein

```shell
DATADIR=/yourUnlabeledDataDir
DATA_BIN=/yourDataBinDir

# Canonicalize all SMILES
python preprocess/canonicalize.py $DATADIR/train.mol --workers 40 \
  --output-fn $DATADIR/train.mol.can

# Tokenize all SMILES
python preprocess/tokenize_re.py $DATADIR/train.mol.can --workers 40 \
  --output-fn $DATADIR/train.mol.can.re 

# Tokenize all protein sequence
python preprocess/add_space.py $DATADIR/train.pro --workers 40 \
  --output-fn $DATADIR/train.pro.addspace

# You should also canonicalize and tokenize the valid set in the same way.

# Binarize the data
fairseq-preprocess \
    --only-source \
    --trainpref $DATADIR/train.mol.can.re \
    --validpref $DATADIR/valid.mol.can.re \
    --destdir $DATA_BIN/molecule \
    --workers 40 \
    --srcdict preprocess/dict.mol.txt
    
fairseq-preprocess \
    --only-source \
    --trainpref $DATADIR/train.pro.addspace \
    --validpref $DATADIR/valid.pro.addspace \
    --destdir $DATA_BIN/protein \
    --workers 40 \
    --srcdict preprocess/dict.pro.txt

# Copy dict to DATA_BIN for evaluation
cp $DATADIR/dict.mol.txt $DATA_BIN
cp $DATADIR/dict.pro.txt $DATA_BIN
```
#### Paired DTI data

```shell
DATADIR=/yourPairedDataDir
DATA_BIN=/yourDataBinDir/bindingdb(davis or kiba)

# Canonicalize all SMILES
python preprocess/canonicalize.py $DATADIR/train.mol --workers 40 \
  --output-fn $DATADIR/train.mol.can

# Tokenize all SMILES
python preprocess/tokenize_re.py $DATADIR/train.mol.can --workers 40 \
  --output-fn $DATADIR/train.mol.can.re 

# Tokenize all protein sequence
python preprocess/add_space.py $DATADIR/train.pro --workers 40 \
  --output-fn $DATADIR/train.pro.addspace

# You should also canonicalize and tokenize the valid set in the same way.

# Binarize the data
fairseq-preprocess \
    --only-source \
    --trainpref $DATADIR/train.mol.can.re \
    --validpref $DATADIR/valid.mol.can.re \
    --destdir $DATA_BIN/input0 \
    --workers 40 \
    --srcdict $DATADIR/dict.mol.txt

fairseq-preprocess \
    --only-source \
    --trainpref $DATADIR/train.pro.addspace \
    --validpref $DATADIR/valid.pro.addspace \
    --destdir $DATA_BIN/input1 \
    --workers 40 \
    --srcdict $DATADIR/dict.pro.txt

mkdir -p $DATA_BIN/label

cp $DATADIR/train.label $DATA_BIN/label/train.label
cp $DATADIR/valid.label $DATA_BIN/label/valid.label
```

### Train Baseline

```shell
DATA_BIN=/yourDataBinDir/bindingdb(davis or kiba)
FAIRSEQ=$pwd
SAVE_PATH=/yourCkptDir
TOTAL_UPDATES=200000 # Total number of training steps
WARMUP_UPDATES=10000 # Warmup the learning rate over this many updates
PEAK_LR=0.00005       # Peak learning rate, adjust as needed
BATCH_SIZE=4		# Batch size
UPDATE_FREQ=4       # Increase the batch size 4x


python $FAIRSEQ/train.py --task dti_separate $DATA_BIN \
    --num-classes 1 --init-token 0 \
    --max-positions-molecule 512 --max-positions-protein 1024 \
    --save-dir $SAVE_PATH \
    --encoder-layers 12 \
    --criterion dti_separate --regression-target \
    --batch-size $BATCH_SIZE --update-freq $UPDATE_FREQ --required-batch-size-multiple 1 \
    --optimizer adam --weight-decay 0.1 --adam-betas '(0.9,0.98)' --adam-eps 1e-06 \
    --lr-scheduler polynomial_decay --lr $PEAK_LR --warmup-updates $WARMUP_UPDATAE --total-num-update $TOTAL_UPDATES \
    --clip-norm 1.0  --max-update $TOTAL_UPDATES \
    --arch roberta_dti_separate_encoder --dropout 0.1 --attention-dropout 0.1 \
    --skip-invalid-size-inputs-valid-test \
    --fp16 \
    --shorten-method truncate \
    --find-unused-parameters | tee -a ${SAVE_PATH}/training.log
```



### Train Ours

```shell
DATA_BIN=/yourDataBinDir
FAIRSEQ=$pwd
SAVE_PATH=/yourCkptDir
TOTAL_UPDATES=200000 # Total number of training steps
INTERVAL_UPDATES=1000 # Validate and save checkpoint every N updates
WARMUP_UPDATES=10000 # Warmup the learning rate over this many updates
PEAK_LR=0.0001       # Peak learning rate, adjust as needed
BATCH_SIZE=4		# Batch size
UPDATE_FREQ=8       # Increase the batch size 8x
MLMW=2 				# MLM loss weight

python $FAIRSEQ/train.py --task dti_mlm_regress_pretrain $DATA_BIN \
    --num-classes 1 --init-token 0 \
    --max-positions-molecule 512 --max-positions-protein 1024 \
    --save-dir $SAVE_PATH \
    --encoder-layers 12 \
    --criterion dti_mlm_regress_pretrain --regression-target \
    --batch-size $BATCH_SIZE --update-freq $UPDATE_FREQ --required-batch-size-multiple 1 \
    --optimizer adam --weight-decay 0.01 --adam-betas '(0.9,0.98)' --adam-eps 1e-06 \
    --lr-scheduler polynomial_decay --lr $PEAK_LR --warmup-updates $WARMUP_UPDATES --total-num-update $TOTAL_UPDATES \
    --clip-norm 1.0  --max-update $TOTAL_UPDATES \
    --arch roberta_dti_mlm_regress --dropout 0.1 --attention-dropout 0.1 \
    --skip-invalid-size-inputs-valid-test \
    --fp16 \
    --shorten-method truncate \
    --use-2-attention --find-unused-parameters --ddp-backend no_c10d \
    --validate-interval-updates $INTERVAL_UPDATES \
    --save-interval-updates $INTERVAL_UPDATES \
    --best-checkpoint-metric loss_regress_mse \
    --mlm-weight-0 $MLMW --mlm-weight-1 $MLMW --mlm-weight-paired-0 $MLMW --mlm-weight-paired-1 $MLMW | tee -a ${SAVE_PATH}/training.log
```
## Evaluation

```shell
python $FAIRSEQ/test.py \
	--checkpoint yourCheckpointPath \
	--data-bin $DATA_BIN \
	--test-data yourTestSetPath \
	--output-fn yourResultPath
```
