To the reviewer,

Thank you for taking the time to review our submission!

This zip file contains the code used to generate the results in the main paper.

# Dependencies

 - pytorch, pytorch-lightning, numpy.
 - You can find exact requirements in `requirements.txt`

# Setup

In a virtual environment, run `python setup.py develop`.

# Generating Data

You will need to generate the data first. To do that, use the `generate_data.py` script. You
will need a copy of the [gSCAN compositional splits](https://github.com/LauraRuis/groundedSCAN/blob/master/data/compositional_splits.zip).

Invoke it like this:

    python scripts/generate_data.py
           --gscan-dataset path/to/compositional_splits/dataset.txt
           --output-directory data/metalearn_sample_environments
           --generate-mode baseline


There are a few different `--generate-mode` options:

 - `baseline`: No-metalearning, to be used with `train_transformer.py` and `train_vilbert.py`
 - `metalearn_allow_any`: Metalearning with oracle instructions and actions, same as **Expert Heuristic** in the paper.
 - `metalearn_random_instructions_same_layout_allow_any`: Metalearning with oracle instructions and actions, but all valid instructions are generated and then selected randomly to form the supports. Same as **Expert Random** in the paper.
 - `metalearn_find_matching_instruction_demos_allow_any`: Meta-learning with heuristic generated instructions and actions for a query, but each support input is solved in some state found in the training data for that input, same as **Expert Other states**

## Generating and Retrieving (GandR)

We have our own implementation of Generate-and-Retrieve (Zemlyanski et al 2022.).

Dependencies are `faiss`, `torch` and `numpy`.

To generate data using this method, you can use the following script:

    python generate_data_imagine_trajectories_gandr.py \ --training-data path/to/baseline/data  \
    --dictionary path/to/baseline/data/dictionary.pb \
    --device cuda \
    --batch-size 128 \
    --data-output-directory path/to/gandr/output \  --load-transformer-model transformer_weights.pb \ --load-state-autoencoder-transformer state_autoencoder.pb \ --save-state-encodings path/to/cached/state/encodings \
    --hidden-size 32 \
    --seed 3 \
    --transformer-iterations 150000 \
    --only-splits train

Note that unlike in the paper, this code can also include compressed
state information in the index. We found that this has not worked
very well, so we omitted it from the paper, as it was not a
good baseline. To include the state information, use `--include-state`.

Once the data generation is complete, the generated examples
and their supports with GandR goes into `/path/to/gandr/output`.

## Learning to Generate Data (DemoGen)

To generate data similar to DemoGen, there are a few steps to be follows.

First, you need to train a regular encoder-decoder Transformer
on the gSCAN baseline data. We suggest using seed 6.

    python scripts/train_transformer.py \
    --train-demonstrations data/baseline \
    --valid-demonstraitons-directory data/baseline \
    --dictionary data/baseline/dictionary.pb \
    --seed 6 \
    --train-batch-size 128 \
    --iterations 300000 \
    --version 100 \
    --enable-progress

After that, you can run the data generation process to
learn both the Masked Language Model and generate the support
sets using the transformer.

    python generate_data_imagine_trajectories.py \
    --data-directory path/to/gscan/baseline/data  \
    --seed 0 \
    --batch-size 64 \
    --device cuda \
    --data-output-directory gen_gscan \
    gscan \
    --save-mlm-model gscan_mlm.pb \
    --mlm-train-iterations 100000 \
    --save-clip-model gscan_clip.pb \
    --clip-train-iterations 100000 \
    --load-transformer-model path/to/saved/transformer/checkpoint.ckpt


The saved data will go into `gen_gscan`, from which you
can use it for `--train-demonstrations` and `--valid-demonstrations`.

# Training the models

To train the models, use something like:

    python scripts/train_meta_seq2seq_transformer.py \
    --train-demonstrations data/metalearn/train.pb \
    --valid-demonstrations data/metalearn/valid \
    --dictionary data/baseline/dictionary.pb \
    --seed 0
    --train-batch-size 32 \
    --valid-batch-size 32 \
    --batch-size-mult 4 \
    --iterations 100 \
    --version 100 \
    --enable-progress

You might want to use a large `--batch-size-mult` to get large effective batch sizes like in the paper.

Logs (both tensorboard and csv logs) are automatic and go to `logs/gscan_s_{seed}_m_{model_name}_it_{iterations}_b_{effective_batch_size}_d_{dataset_name}_t_{tag}_drop_{dropout}/{model_name}_l_{layers}_h_{heads}_d_{embed_dim}/{dataset_name}/{seed}/lightning_logs/version_{version}`

# Analyzing the results and reproducing the Tables in the main paper.

Assuming that you run over seeds 0 through 9
then you can run the `analyze_results.py` script on your `logs` dir with `--logs-dir logs`. This will open all the
logs, exclude the worst seeds and generate the tables.

    python scripts/analyze_results.py --logs-dir path/to/logs

# Analyzing the generated datasets

To reproduce Table 1 in the main paper (showing the properties
of the generated demonstrations), you can run the following script:

    python analyze_generated_datasets.py \
    --data-directory path/to/directory/with/generated/datasets --datasets i2g gandr metalearn_allow_any metalearn_find_matching_instruction_demos_allow_any metalearn_random_instructions_same_layout_allow_any


This will spend a while loading the datasets and performing
the analysis, then print the one table per gSCAN split to
the stdout.

# Analyzing the correctness of generated demonstrations

To reproduce Table 2 in the main paper, you can run the
following script:

    python analyze_supports_correctness.py \
    --data-directory path/to/demonstrations \
    --dictionary path/to/demonstrations/dictionary.pb

The script will evaluate all of the generated instructions
using an oracle function with access to the gSCAN environment
and compare the generated actions with the oracle actions
to determine their correctness.

# Generating the example plots of the supports (Figure 4 in the appendix)

To generate this figure, use the following script:

    python analyze_generate_example_supports_drawing.py \
    --dataset-name name_of_dataset \
    --data-directory path/to/demonstrations \
    --dictionary path/to/demonstrations/dictionary.pb \
    --img-output-directory path/to/imgs \
    --split SPLIT \
    --index INDEX

This generates both the PDF for the environment layout
and also the tikz code used to display one example and its
corresponding supports (along with their relevance, validity
and correctness).

# Performing the failure case analysis (in the Appendix)

This can all be found in the `analyze_failure_cases.py` script. To run this you will need a
trained meta-seq2seq model and transformer model.

    python scripts/analyze_failure_cases.py
    --compositional-splits path/to/gscan/compositional_splits/dataset.txt
    --metalearn-data-directory data/metalearn
    --baseline-data-directory data/baseline
    --meta-seq2seq-checkpoint path/to/metaseq2seq.ckpt
    --transformer-checkpoint path/to/transformer.ckpt

The plots, `comparison_edit_distance_mistakes.pdf`, `num_pulls_vs_edit_distance.pdf` and `pulls_vs_edit_distance_violinplot.pdf` get saved in the current directory.
