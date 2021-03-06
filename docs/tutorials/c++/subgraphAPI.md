## Subgraph API

The subgraph API has been proposed and implemented as the default mechanism for integrating backend libraries to MXNet. The subgraph API is a very flexible interface. Although it was proposed as an integration mechanism, it has been used as a tool for manipulating NNVM graphs for graph-level optimizations, such as operator fusion.

The subgraph API works as the following steps:

* Search for particular patterns in a graph.
* Group the operators/nodes with particular patterns into a subgraph and shrink the subgraph into a single node.
* Replace the subgraph in the original graph with the subgraph node.

The figure below illustrates the subgraph mechanism.

![](https://raw.githubusercontent.com/dmlc/web-data/master/mxnet/tutorials/subgraph/subgraph.png)

The subgraph API allows the backend developers to customize the subgraph mechanism in two places:

* Subgraph searching: define a subgraph selector to search for particular patterns in a computation graph.
* Subgraph node creation: attach an operator to run the computation in the subgraph. We can potentially manipulate the subgraph here.


The following is a demonstration of how the subgraph API can be applied to a simple task. Refer to the previous figure for an overview of the process. That is, replacing `Convolution` and `BatchNorm` with the conv_bn.

The first step is to define a subgraph selector to find the required pattern. To find a pattern that has `Convolution` and `BatchNorm`, we can start the search on the node with `Convolution`. Then from the `Convolution` node, we search for `BatchNorm` along the outgoing edge.

```C++
class SgSelector : public SubgraphSelector {
 public:
  SgSelector() {
    find_bn = false;
  }
  bool Select(const nnvm::Node &n) override {
    // Here we start on the Convolution node to search for a subgraph.
    return n.op() && n.op()->name == "Convolution";
  }
  bool SelectInput(const nnvm::Node &n, const nnvm::Node &new_node) override {
    // We don't need to search on the incoming edge.
    return false;
  }
  bool SelectOutput(const nnvm::Node &n, const nnvm::Node &new_node) override {
    // We search on the outgoing edge. Once we find a BatchNorm node, we won't
    // accept any more BatchNorm nodes.
    if (new_node.op() && new_node.op()->name == "BatchNorm" && !find_bn) {
      find_bn = true;
      return true;
    } else {
      return false;
    }
  }
  std::vector<nnvm::Node *> Filter(const std::vector<nnvm::Node *> &candidates) override {
    // We might have found a Convolution node, but we might have failed to find a BatchNorm
    // node that uses the output of the Convolution node. If we failed, we should skip
    // the Convolution node as well.
    if (find_bn)
      return candidates;
    else
      return std::vector<nnvm::Node *>();
  }
 private:
  bool find_bn;
};
```

The second step is to define a subgraph property to use the subgraph selector above to customize the subgraph searching. By defining this class, we can also customize subgraph node creation. When customizing node creation, we can specify what operator to run the subgraph on the node. In this example, we use `CachedOp`, which itself is a graph executor, to run the subgraph with `Convolution` and `BatchNorm`. In practice, it's most likely that we use a single operator from a backend library to replace the two operators for execution.

```C++
class SgProperty : public SubgraphProperty {
 public:
  static SubgraphPropertyPtr Create() {
    return std::make_shared<SgProperty>();
  }
  nnvm::NodePtr CreateSubgraphNode(
      const nnvm::Symbol &sym, const int subgraph_id = 0) const override {
    // We can use CachedOp to execute the subgraph.
    nnvm::NodePtr n = nnvm::Node::Create();
    n->attrs.op = Op::Get("_CachedOp");
    n->attrs.name = "ConvBN" + std::to_string(subgraph_id);
    n->attrs.subgraphs.push_back(std::make_shared<nnvm::Symbol>(sym));
    std::vector<std::pair<std::string, std::string> > flags{{"static_alloc", "true"}};
    n->attrs.parsed = CachedOpPtr(new CachedOp(sym, flags));
    return n;
  }
  SubgraphSelectorPtr CreateSubgraphSelector() const override {
    return std::make_shared<SgSelector>();
  }
};
```

After defining the subgraph property, we need to register it.

```C++
MXNET_REGISTER_SUBGRAPH_PROPERTY(SgTest, SgProperty);
```

After compiling this subgraph mechanism into MXNet, we can use the environment variable `MXNET_SUBGRAPH_BACKEND` to activate it.

```bash
export MXNET_SUBGRAPH_BACKEND=SgTest
```

This tutorial shows a simple example of how to use the subgraph API to search for patterns in an NNVM graph.
Intested users can try different pattern matching rules (i.e., define their own `SubgraphSelector`) and
attach different operators to execute the subgraphs.

<!-- INSERT SOURCE DOWNLOAD BUTTONS -->
