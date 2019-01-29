
## 新年使用LSTM智能作诗送祝福

### LSTM介绍
序列化数据即每个样本和它之前的样本存在关联，前一数据和后一个数据有顺序关系。深度学习中有一个重要的分支是专门用来处理这样的数据的——循环神经网络。循环神经网络广泛应用在自然语言处理领域，今天我们带你从一个实际的例子出发，介绍神经网络一个重要的改进算法模型，LSTM。本文章不对LSTM的原理进行深入，想详细了解LSTM的可以参考下面的[文章链接](https://www.jianshu.com/p/9dc9f41f0b29)。本文重点从古诗词自动生成的实例出发，一步一步带你从数据处理到模型搭建，再到训练出古诗词生成模型，最后实现从古诗词自动生成新春祝福诗词。

### 数据处理

我们使用76748首古诗词作为数据集，数据集下载链接[]，原始的古诗词的存储形式如下：
![image](https://user-images.githubusercontent.com/43362551/51824023-221ea180-231c-11e9-8577-6595844d752f.png)
原始的古诗词是文本的形式，是一种符号的形式，无法直接进行机器学习，所以我们第一步需要把文本信息转换为数据形式，这种转换方式就叫词嵌入(word embedding)，我们采用一种常用的词嵌套(word embedding)算法-Word2vec对古诗词进行编码。关于Word2Vec这里不详细讲解，有兴趣的可以参考下面的[文章链接](https://zhuanlan.zhihu.com/p/26306795)。在词嵌套过程中，为了避免最终的分类数过于庞大，可以选择去掉出现频率较小的字，比如可以去掉只出现过一次的字。Word2vec算法经过训练后会产生一个模型文件，这样我们就可以应用这个Word2Vec模型对古诗词文本进行词嵌套编码。

经过第一步的处理已经把古诗词词语转换为可以机器学习建模的数字形式，因为我们采用循环神经网络进行古诗词生成，所以还需要构建输入到输出的映射处理。例如：
“[长河落日圆]”作为train_data，而相应的train_label就是“长河落日圆]]”，也就是
“[”->“长”，“长”->“河”，“河”->“落”，“落”->“日”，“日”->“圆”，“圆”->“]”，“]”->“]”，这样子先后顺序一一对相。这也是循环神经网络的一个重要的特征。
这里的“[”和“]”是开始符和结束符，用于生成古诗的开始与结束标记。

总结一下数据处理的步骤：
- 读取原始的古诗词文本，统计出所有不同的字，使用 Word2Vec 算法进行对应编码；
- 对于每首诗，将每个字、标点都转换为字典中对应的编号，构成神经网络的输入数据 train_data；
- 将输入数据左移动构成输出标签 train_label；

经过数据处理后我们得到以下数据文件： 
- poems_edge_split.txt：原始古诗词文件，按行排列，每行为一首诗词；
- vectors_poem.bin：利用 Word2Vec训练好的词向量模型，以</s>开头，按词频排列，去除低频词；
- poem_ids.txt：按输入输出关系映射处理之后的语料库文件；
- rhyme_words.txt： 押韵词存储，用于押韵诗的生成；

在提供的源码中已经提供了以上四个数据文件放在data文件夹下，数据处理代码见 data_loader.py 文件，[源码链接]()


### 模型构建及训练
这里我们使用2层的LSTM框架，每层有128个隐藏层节点，我们使用tensorflow.nn模块库来定义网络结构层，其中RNNcell是tensorflow中实现RNN的基本单元，是一个抽象类，在实际应用中多用RNNcell的实现子类BasicRNNCell或者BasicLSTMCell，BasicGRUCell；如果需要构建多层的RNN，在TensorFlow中，可以使用tf.nn.rnn_cell.MultiRNNCell函数对RNNCell进行堆叠。模型网络的第一层要对输入数据进行 embedding，可以理解为数据的维度变换，经过两层LSTM后，接着softMax得到一个在全字典上的输出概率。
模型网络结构如下：
![image](https://user-images.githubusercontent.com/43362551/51891576-8142eb80-23da-11e9-84c4-66ffdf971818.png)
训练时可以定义batch_size的值，是否进行dropout，为了结果的多样性，训练时在softmax输出层每次可以选择topK概率的字符作为输出。训练时可以使用tensorboard 对网络结构和训练过程可视化展示。这里推荐大家一个在线人工智能建模平台[momodel.cn](momodel.cn)，带有完整的Python和机器学习框架运行环境，并且有免费的GPU可以使用，大家可以训练的时候可以在这个平台上试一下。

定义网络的类的程序代码如下：
``` python

class CharRNNLM(object):
    def __init__(self, is_training, batch_size, vocab_size, w2v_model,
                 hidden_size, max_grad_norm, embedding_size, num_layers,
                 learning_rate, cell_type, dropout=0.0, input_dropout=0.0, infer=False):
        self.batch_size = batch_size
        self.hidden_size = hidden_size
        self.vocab_size = vocab_size
        self.max_grad_norm = max_grad_norm
        self.num_layers = num_layers
        self.embedding_size = embedding_size
        self.cell_type = cell_type
        self.dropout = dropout
        self.input_dropout = input_dropout
        self.w2v_model = w2v_model

        if embedding_size <= 0:
            self.input_size = vocab_size
            self.input_dropout = 0.0
        else:
            self.input_size = embedding_size

        # 输入和输入定义
        self.input_data = tf.placeholder(tf.int64, [self.batch_size, self.num_unrollings], name='inputs')
        self.targets = tf.placeholder(tf.int64, [self.batch_size, self.num_unrollings], name='targets')

        # 根据定义选择不同的循环神经网络内核单元
        if self.cell_type == 'rnn':
            cell_fn = tf.nn.rnn_cell.BasicRNNCell
        elif self.cell_type == 'lstm':
            cell_fn = tf.nn.rnn_cell.LSTMCell
        elif self.cell_type == 'gru':
            cell_fn = tf.nn.rnn_cell.GRUCell

        params = dict()
        if self.cell_type == 'lstm':
            params['forget_bias'] = 1.0
        cell = cell_fn(self.hidden_size, **params)

        cells = [cell]
        for i in range(self.num_layers-1):
            higher_layer_cell = cell_fn(self.hidden_size, **params)
            cells.append(higher_layer_cell)

        # 训练时是否进行 Dropout
        if is_training and self.dropout > 0:
            cells = [tf.nn.rnn_cell.DropoutWrapper(cell, output_keep_prob=1.0-self.dropout) for cell in cells]

        # 对lstm层进行堆叠
        multi_cell = tf.nn.rnn_cell.MultiRNNCell(cells)

        # 定义网络模型初始状态
        with tf.name_scope('initial_state'):
            self.zero_state = multi_cell.zero_state(self.batch_size, tf.float32)
            if self.cell_type == 'rnn' or self.cell_type == 'gru':
                self.initial_state = tuple(
                        [tf.placeholder(tf.float32,
                            [self.batch_size, multi_cell.state_size[idx]],
                            'initial_state_'+str(idx+1)) for idx in range(self.num_layers)])
            elif self.cell_type == 'lstm':
                self.initial_state = tuple(
                        [tf.nn.rnn_cell.LSTMStateTuple(
                            tf.placeholder(tf.float32, [self.batch_size, multi_cell.state_size[idx][0]],
                                          'initial_lstm_state_'+str(idx+1)),
                            tf.placeholder(tf.float32, [self.batch_size, multi_cell.state_size[idx][1]],
                                           'initial_lstm_state_'+str(idx+1)))
                            for idx in range(self.num_layers)])

        # 定义 embedding 层
        with tf.name_scope('embedding_layer'):
            if embedding_size > 0:
                # self.embedding = tf.get_variable('embedding', [self.vocab_size, self.embedding_size])
                self.embedding = tf.get_variable("word_embeddings",
                    initializer=self.w2v_model.vectors.astype(np.float32))
            else:
                self.embedding = tf.constant(np.eye(self.vocab_size), dtype=tf.float32)

            inputs = tf.nn.embedding_lookup(self.embedding, self.input_data)
            if is_training and self.input_dropout > 0:
                inputs = tf.nn.dropout(inputs, 1-self.input_dropout)

        # 创建每个切分通道网络层
        with tf.name_scope('slice_inputs'):
            sliced_inputs = [tf.squeeze(input_, [1]) for input_ in tf.split(
                axis = 1, num_or_size_splits = self.num_unrollings, value = inputs)]

        outputs, final_state = tf.nn.static_rnn(
                cell = multi_cell,
                inputs = sliced_inputs,
                initial_state=self.initial_state)
        self.final_state = final_state

        # 数据变换层，把经过循环神经网络的数据拉伸降维
        with tf.name_scope('flatten_outputs'):
            flat_outputs = tf.reshape(tf.concat(axis = 1, values = outputs), [-1, hidden_size])

        with tf.name_scope('flatten_targets'):
            flat_targets = tf.reshape(tf.concat(axis = 1, values = self.targets), [-1])

        # 定义 softmax 输出层
        with tf.variable_scope('softmax') as sm_vs:
            softmax_w = tf.get_variable('softmax_w', [hidden_size, vocab_size])
            softmax_b = tf.get_variable('softmax_b', [vocab_size])
            self.logits = tf.matmul(flat_outputs, softmax_w) + softmax_b
            self.probs = tf.nn.softmax(self.logits)

        # 定义 loss 损失函数
        with tf.name_scope('loss'):
            loss = tf.nn.sparse_softmax_cross_entropy_with_logits(
                    logits = self.logits, labels = flat_targets)
            self.mean_loss = tf.reduce_mean(loss)

        # tensorBoard 损失函数可视化
        with tf.name_scope('loss_montor'):
            count = tf.Variable(1.0, name='count')
            sum_mean_loss = tf.Variable(1.0, name='sum_mean_loss')

            self.reset_loss_monitor = tf.group(sum_mean_loss.assign(0.0),
                                               count.assign(0.0), name='reset_loss_monitor')
            self.update_loss_monitor = tf.group(sum_mean_loss.assign(sum_mean_loss+self.mean_loss),
                                                count.assign(count+1), name='update_loss_monitor')

            with tf.control_dependencies([self.update_loss_monitor]):
                self.average_loss = sum_mean_loss / count
                self.ppl = tf.exp(self.average_loss)

            average_loss_summary = tf.summary.scalar(
                    name = 'average loss', tensor = self.average_loss)
            ppl_summary = tf.summary.scalar(
                    name = 'perplexity', tensor = self.ppl)

        self.summaries = tf.summary.merge(
                inputs = [average_loss_summary, ppl_summary], name='loss_monitor')

        self.global_step = tf.get_variable('global_step', [], initializer=tf.constant_initializer(0.0))
        self.learning_rate = tf.placeholder(tf.float32, [], name='learning_rate')

        if is_training:
            tvars = tf.trainable_variables()
            grads, _ = tf.clip_by_global_norm(tf.gradients(self.mean_loss, tvars), self.max_grad_norm)
            optimizer = tf.train.AdamOptimizer(self.learning_rate)
            self.train_op = optimizer.apply_gradients(zip(grads, tvars), global_step=self.global_step)

```

### 模型训练

