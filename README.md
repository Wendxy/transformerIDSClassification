# transformerIDSClassification
#### An experiment using encoder only transformer (similar to BERT) in order to classify a sliding window of packets for IDS multi-class classificaton. This is inspired by the success of attention based architectures on sequential information and its ability to map hidden relationship between words in LLMs.

### Early version findings

#### Results summary
Current results are yielding an overall multi-class accuracy of 99.97% +- 0.01% with accuracy on individual classes reaching as high as 99.999% (note classes severely underepresented by the dataset are excluded from this summary). More detail and charts can be found in the jupyter notebooks within this repo.

(Note all results and discussions are based on the training detailed in "IDSTransformerLite.ipynb" this is a smaller version of the original which maintains its performance)

#### Dataset
The dataset is from the KDD cup 1999 (https://kdd.ics.uci.edu/databases/kddcup99/kddcup99.html). It contains several million packet instances and has 23 classes with a highly varied class distribution. However since random sampling is currently used for the test train split, errors made on smaller classes are considered negligible (to be updated).

#### Model inputs
Current model inference (to be updated) is done by inputting a sliding window of 100 packets and classifying the window based on the latest packet into the stream. This means in order for the model to perform its classification tasks we need to run the model on every single incoming packet. This incurs significant cost to network latency. Future work will optimise this by instead experimenting with the bidirectional nature of true BERT and instead classify individual packets regardless of their place in the sliding window, meaning the model will only have to be run every interval of x number of packets instead of every individual packet. This flexible approach to the window (which in this case behaves like a GPT input token limit) will allow us to create a function between the length of the sliding window (x), and inference time (t) which can be minimised. In addition to this, no hyperparameter tuning was utilised (left for future work).

#### Discussion of transformer and their advantages and disadvantages
The advantage of this approach compared to others is a transformers ability to intepret the "tokens" (packets) within our sequence (sliding window) all at once, while learning the semantic relationships within packets across time to identify attacks across a wider variety of time frames. In addition to this, its ability to capture the relationship between each individual packets should in theory permit the identification of more complex or unorthodox attack patterns. Furthermore, given the greater flexibility afforded by transformer's complexity, it is more likely that this model will scale better with data compared to smaller parameter models, allowing it to capture even more complex relationships while hitting diminishing returns on data further into training. This complexity however does come at the cost of signficantly higher compute overhead compared to simpler architectures (which we discuss methods to mitigate below).

#### Training behaviour
Current training behaviour reveals a low number of epochs to train (usually ~3 for loss to converge, after which signs of minor overfitting often arise), which is consistent with the number of epochs required to post train large language models. Often it can be seen that validation loss moves "sideways" from the first epoch, however this does not significantly effect final evaluation. Further analysis on confusion matrices and other metrics for **each** epoch (currently only looking at final model results) will be done in order to interpret model behaviour across epochs in these instances. 

#### Training Compute Performance
This model was trained on a 4070 ti super with 16gb vram and a batch size of 32. With this set up each epoch takes ~6min to complete, this may be sped up significantly as GPU usage analysis shows potential room to increase batch size. No measurement was done regarding FLOPS.

#### Inference Compute Performance
Current inference time is between 0.6-0.8ms for a sliding window of 100 packets (on GPU with batch size = 1 during eval). If we were using the ideal window input style detailed above where we only read in intervals of 100 (instead of every single one) this would be sufficient assuming packets are 1500 bytes (average packet size) and we have a 67mbps link (average speed in Australia). With these assumed conditions every 0.70ms ~4 packets will be sent allowing model inference to keep ahead of packet stream with significant headroom, however this calculation does not consider outside of this packet stream condition (such as with consistently minimum packet sizes instead of 1500 bytes) and it does not consider hardware constrained environments where the current model may still be too slow. In saying this, significant performance gains ( ~3x faster and ~4x smaller) potentially can be made via quantizing the model and / or ablative parameter reduction. (note: I've just reduced the number of certain layers by half without any impact to accuracy).

Assuming approximately 5500 packets per second being sent, at our maximum experimental accuracy of 99.98% the model only misclassifies 1 packet out of the 5500 per second (using x ~ geo(0.0002) expectation and assuming good generalization - note old dataset).

#### Changes in the Lite model
Three main changes were made to achieve the light model. First, the number of attention heads and layers were halved and the embedding dimension was reduced by a factor of 4. While this decrease in model parameters and lower granularity from the smaller embedding space would intuitively decrease the model's performance, it infact improved by model's performance by a few percentage decimal places. I hypothesise this is due to the simpler nature of the packet stream (at least this 25 year old dataset's packets), and the fact that the original configuration was overkill for the task. The excess of layers and embedding dimensions would lead to underutilisation of certain components, and the larger parameter size may have fitted to noise rather than legitimate patterns. The speed metrics within this brief report are based on the Lite model's performance. 

The reduction in the model's precision via its simplification creates optimism that further decreases in precision using quantization will not have a significant impact on model accuracy while providing extremely significant performance gains.

#### Conclusion
While transformer based architectures inherently require heavier compute than simpler architectures, with the correct optimizations of sliding window size, model quantizing, packet sniffer hardware optimization (to be compatible with quantised models), and empirically guided ablations, this nascent architecture is a promising future facing paradigm in IDS especially as attack patterns become significantly more complex and varied such that simpler classification models can no longer keep up. 
---
## Quantised Model Experiment



