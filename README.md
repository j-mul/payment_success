# payment_success repo - Predicting Successful Transactions

In this task we are interested in aiming to predict whether a transaction is likely to be successful. Given the data being at the transaction level, I took the approach that we are primarily interested in scoring transactions as they come in. 

*As an aside; given more time and a better understanding of the data & business case, I would be interested in examining how we may be able to improve the recall of this approach by rescoring transactions at certain intervals (Initiated, Authorised, Executing). My thinking is that the business may be more interested in detecting potential payment failure earlier in the process, perhaps to hold out on immediat/upfront settlement, in which case it would be less beneficial to rescore. Additionally, there is the resource cost implication, but still it would be an interesting piece of analysis.*


The chosen approach to the problem has a implications for the data fields we could utilise within this set.

- Timestamp features are unhelpful in predicting this problem, as we would not have meaningful data at inception of a transaction for these fields. Notwithstanding data sparsity/integrity for these fields.
- Failure stage is not helpful for prediction as this information is not present at inception.
- Failure Reason is interesting from a process point of view, and understanding why we may see failed transactions, though is unable to be used for prediction as obviously it is also not present at inception - we learn this post failure.
- Lastly 'id' being the transaction identifier, obviously isn't considered for prediction.

This leaves us with the following raw data we can leverage for prediction:
- bank_id
- currency
- api_version
- vertical 
- connectivity_type
- amount_in_currency
- country_id 
- customer_id

Given the lack of data dictionary with this task, I was limited in my understanding of how to engineer features manually to enhance their predictive power. Though certain generic steps were taken to make life a bit easier for the model!

Nulls were spotted in the API version & Vertical fields, these were associated with authorization failures probably due to technical problems with API calls. The choice was made to omit these rows as while they are associated with payment failure, they could be easily detected without need for a ML solution if my assumpmtions hold true.

The customer_id field was also cleaned to wrap id's with less than 100 obeservations in the data to a new category 'rare_customer'. This is to make the process of encoding these categories in the model a little more robust.

### Model

The model I chose to use was a CatBoost classifier. This model natively handles categorical features in their string format and encodes them using ordered target based encoding. As we have high cardinality in the categorical features 'bank_id' & 'customer_id', this was ideal for our problem, as we lack the ability to use One Hot encoding (the curse of dimensionality). In addition to this CatBoost leverages the performative capabilities of gradient boosting models, performs well with relatively little hyperparameter tuning and can be trained quickly on GPU. The target for the classifier was created as a binary feature 'target' - 1 if payment was executed else 0. The problem statement asks if we can predict the likelihood of a payment being successful, the ouput score of a classifier model can tell us this.

The model was trained using a small hyperparameter sweep for tuning and 4 cv folds to mitigate overfitting.

### Results

The model displays no evidence of overfit, demonstrated by the similarity in training and testing metrics which is positive. In brief the model predicts the majority class well, which is what we tend to experience in datasets with moderate class imbalance such as this. What we are more interested in is it's ability to detect the minority class, in this case payment failures. The model with default classification threshold of .5 achieves recall of .17 on the negative class i.e. correctly identifies 17% of all payment failures, with a precision of .65 i.e. 65% of predicted failures are true failures. Threshold adjustment will allow the business stakeholders to select a desired level of precision - by increasing the threshold we will see precision increase at the expense of recall, and vice versa. If we want to capture more true failures we could adjust the threshold downwards, but we would see more false negatives.

The aim of this model as outlined in the brief was to assess how likely a payment was to be successful. Looking at the calibration curve on the test set, we can see that the model performs pretty well in this task, with the results being w=very close to perfect calibration across all deciles. This means we can trust the probability scores output from this calssifier as a decent proxy for likelihood of success.

### Deployment

I would envision this model being deployed for online prediction, with each transaction being scored as it is initiated. I invision this being used to put a hold on advanced payment of transactions with a low likelihood of success. For this to be achieved, we need a very low latency architecture, as payments usually settle/fail within seconds of being initiated.

In terms of how this is achieved, we would need to work with engineering teams to develop streamlined ETL payloads and microservices for model orchestration with this as the goal. Model serialization could be optimised with speed of inference in mind. In addition, we would want to experiment with various AWS EC2 instance specifications to strike a balance between performance and cost of inference.
