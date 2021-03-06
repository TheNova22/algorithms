<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [k-d 树](#k-d-%E6%A0%91)
  - [k-d 树的创建](#k-d-%E6%A0%91%E7%9A%84%E5%88%9B%E5%BB%BA)
  - [k-d 树的搜索](#k-d-%E6%A0%91%E7%9A%84%E6%90%9C%E7%B4%A2)
    - [推荐资料](#%E6%8E%A8%E8%8D%90%E8%B5%84%E6%96%99)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## k-d 树

所谓 k-d 树，其实就是 K-维树，在 K 维度的空间，每次选择一个维度 D 来里进行切分，直到无法分割为止。比如有一堆 K 维度的数据，我们每次针对某一维度，利用其中位数进行分割成两个部分。然后在这两个分割结果中，分别改变维度继续切分。

最简单的例子就是在空间直角坐标系中的切分了，只有 x/y 两个维度。交替在 x, y 轴上进行切分。

例如有如下数据集：`(1, 7), (2, 3), (1, 5), (2, 2), (9, 6), (1, 8), (8, 1)`，可以计算出，在 x 轴上的方差大于在 y 轴上的方差，因此首先选择 x 轴进行分割。

1. 第一次分割，沿 x 轴选取所有坐标的中位数 2。x 大于 2 的点划分到右子树，x 小于 2 的点划分到左子树。
2. 第二次分割，分别对左右两子树通过 y 轴进行划分。
  1. 对于左侧，在 y 轴上的数字有 5, 7, 8，因此通过 y = 7 进行划分
  2. 对于右侧，只剩下两个点 (8, 1) 和 (9, 6)，通过 y = 6 进行划分
3. 每次划分交替使用的轴，直至各个区域内不可再分为止

### k-d 树的创建

将上述方法推广到 K 维空间，每次对 第 d 维空间进行划分。而实际上，为了防止数据在某个维度上密集分布而在其他维度上稀疏的情况（可以想象一根圆柱，其长 x、宽 y、高 z 为各个维度，显然数据在 x 维度上拉伸的较长，但在另外两个维度上密集分布），我们每次划分应该计算各个维度区间内的方法，然后在拥有最大方差的维度上进行划分：

1. 在 K 维数据集合中选择具有最大方差的维度 d，然后在该维度上选择中值对该数据集进行划分，得到左右两个子集合
2. 对两个子树重复步骤 1，直至所有子集合都不能再划分为止

```javascript
class Node {
  constructor(options) {
    const {
      point, // 将点转换为数组的格式 (1, 2) => [1, 2]
      dimensional, // 因此可以利用索引来代表不同维度
      parentNode = null,
    } = options;
    this.point = point;
    this.parentNode = parentNode;
    this.dimensional = dimensional;
    this.leftNode = null;
    this.rightNode = null;
    this.visited = false;
  }
}

// 计算方差
const getVariance = (array) => {
  const avg = array.reduce((pre, next) => pre + next, 0) / array.length;
  return array.reduce((pre, next) => Math.pow(next - avg, 2) + pre, 0) / array.length;
};

// 获取中位数所在的索引
// TODO: 使用算法来优化这一过程
const getCentralIndex = (dataset, dimensional) => {
  if (dataset.length <= 1) return 0;
  dataset.sort((pre, current) => pre[dimensional] - current[dimensional]);
  return Math.floor(dataset.length / 2);
};

// 通过最大方差来获取分隔的维度
// TODO: 使用算法来优化这一过程
const getDimensional = (dataset) => {
  const point = dataset[0];
  let dimensional = null;
  let maxVariance = null;

  // i means current dimensional
  for (let i = 0; i < point.length; i += 1) {
    const datas = dataset.map(point => point[i]);
    const variance = getVariance(datas);
    if (!maxVariance || variance > maxVariance) {
      maxVariance = variance;
      dimensional = i;
    }
  }
  return dimensional;
};

const build = (dataset, parentNode = null) => {
  if (!dataset.length) return null;
  const dimensional = getDimensional(dataset);
  const centralIndex = getCentralIndex(dataset, dimensional);
  const left = dataset.slice(0, centralIndex);
  const right = dataset.slice(centralIndex + 1);

  const node = new Node({
    point: dataset[centralIndex],
    dimensional,
    parentNode
  });

  const leftNode = build(left, node);
  const rightNode = build(right, node);

  node.leftNode = leftNode;
  node.rightNode = rightNode;
  return node;
};
```

### k-d 树的搜索

给定一个点 p 和一个 k-d 树，可以搜索到距离 p 点最近的点：

1. 从树的根部开始，依次根据各个节点的划分维度信息进行比较，遍历到树的最底部 - 即确定 p 点被划分的区域

2. 找到该区域内的点（没有的话就使用当前节点），计算两者距离，记录为 D

  2.1. 计算 p 点到当前节点划分维度的距离 D1，比较其和 D 的大小：
    - 如果 D1 < D，则在维度的另一边有可能存在距离更近的点。向上遍历到父节点，然后到达其另外的子节点，寻找其他节点内的元素，计算距离。如果存在点使得距离小于 D，则用新的距离替代 D
    - 如果 D1 > D，则维度的另一边不可能存在更近的点，不做操作。

  2.2. 继续向上回溯，到父节点，重复 2.1 的操作

如果要获取距离指定点最近的 N 个点，则从底部节点开始，先对结果进行填充，把各点存起来；之后如果计算得到距离更小的值，则进行替换。这样即是 k-d 树在 knn 算法(k-nearest neighbors)中的应用。

```javascript
class Node {
  // 根据输入的点，走到当前节点最底部的位置
  bottom(point) {
    if (!this.leftNode && !this.rightNode) return this;

    const splitValue = this.point[this.dimensional];
    const target = point[this.dimensional];

    if (target === splitValue) return this;
    if (target < splitValue) {
      if (!this.leftNode) return this;
      return this.leftNode.bottom(point);
    }
    if (!this.rightNode) return this;
    return this.rightNode.bottom(point);
  }

  // 计算在当前节点时，目标点到当前节点分隔维度的直线距离
  // 以此来断定需不需要遍历节点的子树
  verticalDistance(point) {
    return Math.abs(this.point[this.dimensional] - point[this.dimensional]);
  }
}
```

而我们在其遍历过程中，储存个点和目标点的距离，并筛选出有最小距离的几个点时，可以通过最大二叉堆来优化算法速度。即：

- 已知我们要获取距离目标点最近的 N 个点
- 首先到底树的最底层，记录下最底层该节点的值和距离，构建出一个最大堆。
- 向上遍历，当堆中元素小于 N 个时，所遇见的点都会被收纳到堆中，直至数目为 N 为止。
- 之后，如果遇见新的点，其距离小于堆顶的值，则将最大二叉堆的根部节点出堆，然后新的值入堆

#### 推荐资料

- [一只兔子帮你理解 kNN](https://www.joinquant.com/view/community/detail/a98b7021e7391c62f6369207242700b2)
- [kd 树算法之思路篇](https://www.joinquant.com/post/2627)
- [kd 树算法之详细篇](https://www.joinquant.com/post/2843)
- [Kd Tree算法原理和开源实现代码](https://my.oschina.net/keyven/blog/221792)