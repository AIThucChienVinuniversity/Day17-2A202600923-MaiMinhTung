# Reflection — Day 17

### 1. The flywheel

The most dangerous silent failure would be the dataset curation and decontamination stage between Bronze traces and the final training datasets. If traces are incorrectly labeled or preference pairs are generated from bad responses, the pipeline would still run successfully but produce low-quality training data. This is difficult to notice because there may be no system error. I would detect it by monitoring dataset statistics over time, such as the number of eval examples, preference pairs, error rates, and the ratio of accepted versus rejected pairs. Sudden changes would indicate a potential problem.

### 2. Decontamination

If I skip decontamination, prompts from the evaluation set may also appear in the training data. The model would effectively see test questions during training and memorize the correct answers. As a result, evaluation metrics would become overly optimistic and no longer reflect real-world performance. The model might achieve higher benchmark scores while showing little or no improvement on truly unseen questions. This creates a misleading sense of progress.

### 3. Point-in-time

A dangerous example is fraud detection in financial transactions. Suppose a model predicts whether a transaction is fraudulent. If the training data is joined with a customer's future account balance or future spending behavior, the model gains information that was not available when the transaction occurred. This would lead to unrealistic performance during training and poor results in production. Using an ASOF join ensures that only information available at the event time is used.

### 4. Graph vs Vector

The knowledge graph answers multi-hop questions well, such as “Where does a widget ship from?”. The graph can traverse the relationships widget → accessory → Hanoi fulfillment center and find the answer. Flat chunk retrieval struggles because the relevant facts are stored in different chunks. On the other hand, for a simple question such as “What is the warranty period of a gadget?”, a vector search is usually sufficient and the graph would be unnecessary complexity.
