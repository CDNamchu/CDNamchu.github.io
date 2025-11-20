---
title: "Graph Neural Networks for Code Analysis: A Practical Guide"
date: 2025-11-18
categories: [AI, Blog]
tags: [Machine Learning, Graph Neural Networks, Deep Learning, PyTorch]
---

# Graph Neural Networks for Code Analysis: A Practical Guide

Code is more than just text - it's a rich, structured representation of logic, dependencies, and relationships. Graph Neural Networks (GNNs) provide a powerful framework for understanding and analyzing code structure, enabling applications from bug detection to code recommendation systems.

## Why Graphs for Code?

Traditional approaches to code analysis treat source code as text or token sequences. While this works for some tasks, it misses crucial structural information:

- **Function call relationships** - Who calls whom?
- **Data flow** - How does data move through the program?
- **Control flow** - What's the execution path?
- **Variable dependencies** - Which variables depend on others?

Graphs naturally represent these relationships, making GNNs ideal for code analysis tasks.

## Code Representation as Graphs

### Abstract Syntax Trees (AST)

The most common representation is the Abstract Syntax Tree:

```python
# Example code
def factorial(n):
    if n <= 1:
        return 1
    return n * factorial(n - 1)
```

This becomes a tree where nodes represent code constructs (functions, conditionals, operations) and edges represent hierarchical relationships.

### Control Flow Graphs (CFG)

CFGs represent the flow of execution:

```
[Start] → [Check n <= 1] → [Return 1]
              ↓
        [Calculate n * factorial(n-1)] → [Return result]
```

### Data Flow Graphs (DFG)

DFGs track how data moves through the program:

```
Input 'n' → Used in comparison → Used in multiplication → Return value
```

## Building a GNN for Code Analysis

Let's build a complete system for analyzing Python code using GNNs.

### Step 1: Parse Code to Graph

```python
import ast
import networkx as nx
from torch_geometric.utils import from_networkx

class CodeToGraph:
    def __init__(self):
        self.node_types = {
            'FunctionDef': 0, 'If': 1, 'Return': 2,
            'BinOp': 3, 'Compare': 4, 'Call': 5,
            'Name': 6, 'Constant': 7
        }
    
    def parse_file(self, code_string):
        """Parse Python code and convert to graph"""
        tree = ast.parse(code_string)
        graph = nx.DiGraph()
        
        self._build_graph(tree, graph)
        return self._to_pyg_data(graph)
    
    def _build_graph(self, node, graph, parent_id=None, node_id=0):
        """Recursively build graph from AST"""
        current_id = node_id
        node_type = type(node).__name__
        
        # Add node with features
        graph.add_node(current_id, 
                      node_type=self.node_types.get(node_type, -1),
                      label=node_type)
        
        # Add edge from parent
        if parent_id is not None:
            graph.add_edge(parent_id, current_id, edge_type='ast')
        
        # Process children
        next_id = current_id + 1
        for child in ast.iter_child_nodes(node):
            next_id = self._build_graph(child, graph, current_id, next_id)
        
        return next_id
    
    def _to_pyg_data(self, graph):
        """Convert NetworkX graph to PyTorch Geometric Data"""
        # Add node features
        for node in graph.nodes():
            graph.nodes[node]['x'] = graph.nodes[node]['node_type']
        
        return from_networkx(graph, group_node_attrs=['x'])
```

### Step 2: Enhanced Features

Add more sophisticated features:

```python
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer

class EnhancedCodeFeatures:
    def __init__(self):
        self.vectorizer = TfidfVectorizer(max_features=100)
    
    def extract_features(self, node, code_context):
        """Extract rich features for each node"""
        features = []
        
        # 1. Node type (one-hot encoded)
        node_type_vec = self._one_hot_encode(node.type, num_types=20)
        features.append(node_type_vec)
        
        # 2. Structural features
        features.append([
            node.depth,                    # Depth in AST
            node.num_children,             # Number of children
            node.num_siblings,             # Number of siblings
            int(node.is_leaf)              # Is leaf node
        ])
        
        # 3. Code complexity metrics
        if hasattr(node, 'code_string'):
            features.append([
                len(node.code_string),     # Code length
                node.code_string.count('\n'),  # Lines of code
                self._cyclomatic_complexity(node)  # Complexity
            ])
        
        # 4. Semantic features (embeddings)
        if hasattr(node, 'identifier'):
            name_embedding = self._get_name_embedding(node.identifier)
            features.append(name_embedding)
        
        return np.concatenate(features)
    
    def _cyclomatic_complexity(self, node):
        """Calculate cyclomatic complexity"""
        complexity = 1
        decision_points = ['If', 'While', 'For', 'And', 'Or']
        for child in ast.walk(node.ast_node):
            if type(child).__name__ in decision_points:
                complexity += 1
        return complexity
```

### Step 3: GNN Architecture for Code

```python
import torch
import torch.nn as nn
from torch_geometric.nn import GCNConv, GATConv, global_mean_pool

class CodeAnalysisGNN(nn.Module):
    def __init__(self, num_features, hidden_dim=256, num_classes=10):
        super(CodeAnalysisGNN, self).__init__()
        
        # Graph Attention Networks for learning important relationships
        self.conv1 = GATConv(num_features, hidden_dim, heads=4, dropout=0.3)
        self.conv2 = GATConv(hidden_dim * 4, hidden_dim, heads=4, dropout=0.3)
        self.conv3 = GATConv(hidden_dim * 4, hidden_dim, heads=1, dropout=0.3)
        
        # Batch normalization
        self.bn1 = nn.BatchNorm1d(hidden_dim * 4)
        self.bn2 = nn.BatchNorm1d(hidden_dim * 4)
        
        # Skip connections
        self.skip1 = nn.Linear(num_features, hidden_dim * 4)
        self.skip2 = nn.Linear(hidden_dim * 4, hidden_dim)
        
        # Classifier head
        self.classifier = nn.Sequential(
            nn.Linear(hidden_dim, 128),
            nn.ReLU(),
            nn.Dropout(0.5),
            nn.Linear(128, 64),
            nn.ReLU(),
            nn.Dropout(0.3),
            nn.Linear(64, num_classes)
        )
    
    def forward(self, data):
        x, edge_index, batch = data.x, data.edge_index, data.batch
        
        # First layer with skip connection
        identity = self.skip1(x)
        x = self.conv1(x, edge_index)
        x = self.bn1(x)
        x = torch.relu(x + identity)
        
        # Second layer with skip connection
        identity = x
        x = self.conv2(x, edge_index)
        x = self.bn2(x)
        x = torch.relu(x + identity)
        
        # Third layer
        x = self.conv3(x, edge_index)
        x = torch.relu(x)
        
        # Global pooling
        x = global_mean_pool(x, batch)
        
        # Classification
        return self.classifier(x)
```

### Step 4: Multi-Task Learning

Analyze multiple aspects simultaneously:

```python
class MultiTaskCodeGNN(nn.Module):
    def __init__(self, num_features, hidden_dim=256):
        super(MultiTaskCodeGNN, self).__init__()
        
        # Shared encoder
        self.encoder = nn.ModuleList([
            GATConv(num_features, hidden_dim, heads=4),
            GATConv(hidden_dim * 4, hidden_dim, heads=4),
            GATConv(hidden_dim * 4, hidden_dim, heads=1)
        ])
        
        # Task-specific heads
        self.bug_detector = nn.Linear(hidden_dim, 2)  # Bug or no bug
        self.complexity_predictor = nn.Linear(hidden_dim, 1)  # Complexity score
        self.code_smell_detector = nn.Linear(hidden_dim, 5)  # Different code smells
        
    def forward(self, data):
        x, edge_index, batch = data.x, data.edge_index, data.batch
        
        # Shared encoding
        for conv in self.encoder:
            x = conv(x, edge_index).relu()
        
        x = global_mean_pool(x, batch)
        
        # Multi-task predictions
        return {
            'bug_prediction': self.bug_detector(x),
            'complexity': self.complexity_predictor(x),
            'code_smells': self.code_smell_detector(x)
        }
```

### Step 5: Training Pipeline

```python
from torch_geometric.loader import DataLoader
import torch.nn.functional as F

class CodeAnalysisTrainer:
    def __init__(self, model, device='cpu'):
        self.model = model.to(device)
        self.device = device
        self.optimizer = torch.optim.Adam(model.parameters(), lr=0.001)
        
    def train_epoch(self, train_loader):
        self.model.train()
        total_loss = 0
        
        for batch in train_loader:
            batch = batch.to(self.device)
            self.optimizer.zero_grad()
            
            # Forward pass
            out = self.model(batch)
            
            # Calculate loss
            loss = F.cross_entropy(out, batch.y)
            
            # Backward pass
            loss.backward()
            self.optimizer.step()
            
            total_loss += loss.item()
        
        return total_loss / len(train_loader)
    
    def evaluate(self, test_loader):
        self.model.eval()
        correct = 0
        total = 0
        
        with torch.no_grad():
            for batch in test_loader:
                batch = batch.to(self.device)
                out = self.model(batch)
                pred = out.argmax(dim=1)
                correct += (pred == batch.y).sum().item()
                total += batch.y.size(0)
        
        return correct / total
```

## Real-World Applications

### 1. Bug Detection

Train on repositories with labeled bugs:

```python
# Label: 0 = no bug, 1 = bug present
buggy_code = """
def transfer(balance, amount):
    if balance >= amount:  # Missing return after check
        balance -= amount
    # Bug: no explicit return, might return None
"""

# Train model to detect such patterns
model = CodeAnalysisGNN(num_features=128, num_classes=2)
```

### 2. Code Similarity

Use graph embeddings for clone detection:

```python
class CodeSimilarityGNN(nn.Module):
    def __init__(self, num_features, embedding_dim=128):
        super().__init__()
        self.encoder = CodeAnalysisGNN(num_features, embedding_dim, embedding_dim)
    
    def forward(self, code1, code2):
        emb1 = self.encoder(code1)
        emb2 = self.encoder(code2)
        
        # Cosine similarity
        similarity = F.cosine_similarity(emb1, emb2)
        return similarity

# Use for clone detection
similarity_score = model(graph1, graph2)
if similarity_score > 0.8:
    print("Possible code clone detected!")
```

### 3. Code Completion

Predict next nodes in the graph:

```python
class CodeCompletionGNN(nn.Module):
    def __init__(self, num_features, vocab_size):
        super().__init__()
        self.gnn = CodeAnalysisGNN(num_features, 256, vocab_size)
    
    def predict_next(self, partial_code_graph):
        logits = self.gnn(partial_code_graph)
        return torch.softmax(logits, dim=-1)
```

## Performance Optimization

### 1. Efficient Batching

```python
from torch_geometric.loader import DataLoader

# Use dynamic batching for variable-sized graphs
train_loader = DataLoader(
    dataset, 
    batch_size=32,
    shuffle=True,
    num_workers=4,
    pin_memory=True
)
```

### 2. Model Pruning

```python
import torch.nn.utils.prune as prune

# Prune less important connections
for module in model.modules():
    if isinstance(module, GCNConv):
        prune.l1_unstructured(module, name='weight', amount=0.3)
```

### 3. Knowledge Distillation

Train a smaller student model:

```python
def distillation_loss(student_logits, teacher_logits, labels, temperature=3.0):
    soft_loss = F.kl_div(
        F.log_softmax(student_logits / temperature, dim=1),
        F.softmax(teacher_logits / temperature, dim=1),
        reduction='batchmean'
    ) * (temperature ** 2)
    
    hard_loss = F.cross_entropy(student_logits, labels)
    
    return 0.7 * soft_loss + 0.3 * hard_loss
```

## Evaluation Metrics

```python
from sklearn.metrics import precision_recall_fscore_support, confusion_matrix

def evaluate_comprehensive(model, test_loader):
    all_preds = []
    all_labels = []
    
    model.eval()
    with torch.no_grad():
        for batch in test_loader:
            out = model(batch)
            pred = out.argmax(dim=1)
            all_preds.extend(pred.cpu().numpy())
            all_labels.extend(batch.y.cpu().numpy())
    
    # Calculate metrics
    precision, recall, f1, _ = precision_recall_fscore_support(
        all_labels, all_preds, average='weighted'
    )
    
    print(f"Precision: {precision:.4f}")
    print(f"Recall: {recall:.4f}")
    print(f"F1 Score: {f1:.4f}")
    
    # Confusion matrix
    cm = confusion_matrix(all_labels, all_preds)
    return {'precision': precision, 'recall': recall, 'f1': f1, 'cm': cm}
```

## Challenges and Future Directions

### Current Challenges

1. **Scalability** - Large codebases create huge graphs
2. **Feature engineering** - Choosing the right features is critical
3. **Labeled data** - Need large datasets of labeled code
4. **Interpretability** - Understanding why the model makes predictions

### Future Directions

1. **Transfer learning** - Pre-train on large code corpora
2. **Multi-modal learning** - Combine code structure with comments and documentation
3. **Temporal analysis** - Analyze code evolution over time
4. **Cross-language support** - Unified representations for multiple languages

## Conclusion

Graph Neural Networks offer a powerful paradigm for code analysis, capturing the rich structural information that traditional approaches miss. From bug detection to code generation, GNNs are transforming how we understand and work with code.

The key is representing code appropriately as graphs and designing GNN architectures that can learn meaningful patterns from these structures. As the field evolves, we'll see more sophisticated applications that make developers more productive and codebases more secure.

## Resources

- [PyTorch Geometric Documentation](https://pytorch-geometric.readthedocs.io/)
- [Graph Neural Networks for Code](https://arxiv.org/abs/1711.00740)
- [CodeBERT and Graph-based Models](https://github.com/microsoft/CodeBERT)
- [AST-based Analysis Tools](https://docs.python.org/3/library/ast.html)

---

*Working on GNN applications? Let's discuss architectures and optimization strategies!*
