# DFS(Depth First Search)
> DFS는 한 경로를 끝까지 깊게 탐색한 뒤, 더 이상 진행할 수 없으면 이전 단계로 되돌아와 다른 경로를 탐색하는 방식.
> → 마치 미로를 탐험할 때 한쪽 길을 끝까지 들어갔다가 막히면 되돌아오는 과정과 유사
## 적합한 상황
- 백트래킹
- 퍼즐 탐색 
- 경로 존재 여부 확인

##  구현 방식
### recursion - call stack
```java
public class DFSExample {
    private Map<Integer, List<Integer>> graph = new HashMap<>();
    private Set<Integer> visited = new HashSet<>();

    public void addEdge(int v, int w) {
        graph.computeIfAbsent(v, k -> new ArrayList<>()).add(w);
        graph.computeIfAbsent(w, k -> new ArrayList<>()).add(v); // 무방향 그래프
    }

    public void dfs(int v) {
        if (visited.contains(v)) return;

        visited.add(v);
        System.out.print(v + " ");

        for (int neighbor : graph.getOrDefault(v, new ArrayList<>())) {
            if (!visited.contains(neighbor)) {
                dfs(neighbor);
            }
        }
    }

    public static void main(String[] args) {
        DFSExample g = new DFSExample();
        g.addEdge(1, 2);
        g.addEdge(1, 3);
        g.addEdge(2, 4);
        g.addEdge(3, 5);
        g.addEdge(4, 6);

        System.out.print("DFS 탐색 순서: ");
        g.dfs(1);
    }
}
```
### iteration - stack
```java
public class DFSIterative {
    private final Map<Integer, List<Integer>> graph = new HashMap<>();

    public void addEdge(int v, int w) {
        graph.computeIfAbsent(v, k -> new ArrayList<>()).add(w);
        graph.computeIfAbsent(w, k -> new ArrayList<>()).add(v); // 무방향 그래프
    }

    public void dfs(int start) {
        Set<Integer> visited = new HashSet<>();
        Stack<Integer> stack = new Stack<>();
        stack.push(start);

        while (!stack.isEmpty()) {
            int v = stack.pop();

            if (!visited.contains(v)) {
                visited.add(v);
                System.out.print(v + " ");

                List<Integer> neighbors = graph.get(v);
                if (neighbors != null) {
                    // 역순으로 넣어야 "작은 번호부터 방문" 보장됨
                    Collections.reverse(neighbors);
                    for (int neighbor : neighbors) {
                        if (!visited.contains(neighbor)) {
                            stack.push(neighbor);
                        }
                    }
                }
            }
        }
    }

    public static void main(String[] args) {
        DFSIterative g = new DFSIterative();
        g.addEdge(1, 2);
        g.addEdge(1, 3);
        g.addEdge(2, 4);
        g.addEdge(3, 5);
        g.addEdge(4, 6);

        System.out.print("DFS (스택) 탐색 순서: ");
        g.dfs(1);
    }
}
```
**시간 복잡도**
- 정점 방문: O(V)
- 간선 확인: O(E)
- 전체 시간 복잡도: `𝑂(𝑉 + 𝐸)`

---
# BFS
> 가까운 노드부터 차례대로 탐색하는 방식이다. 시작점에서 1단계 떨어진 노드를 먼저 탐색하고, 그다음 2단계, 3단계… 순으로 탐색한다.
> → 물결이 번지는 것처럼 레벨 단위로 확산되는 탐색이다.

## 적합한 상황
- 최단 경로 탐색
- 네트워크 전파 문제

## 구현 방식
### Queue 이용
```java
public class BFSExample {
    private Map<Integer, List<Integer>> graph = new HashMap<>();

    public void addEdge(int v, int w) {
        graph.computeIfAbsent(v, k -> new ArrayList<>()).add(w);
        graph.computeIfAbsent(w, k -> new ArrayList<>()).add(v); // 무방향 그래프
    }

    public void bfs(int start) {
        Set<Integer> visited = new HashSet<>();
        Queue<Integer> queue = new LinkedList<>();

        visited.add(start);
        queue.offer(start);

        while (!queue.isEmpty()) {
            int v = queue.poll();
            System.out.print(v + " ");

            for (int neighbor : graph.getOrDefault(v, new ArrayList<>())) {
                if (!visited.contains(neighbor)) {
                    visited.add(neighbor);
                    queue.offer(neighbor);
                }
            }
        }
    }

    public static void main(String[] args) {
        BFSExample g = new BFSExample();
        g.addEdge(1, 2);
        g.addEdge(1, 3);
        g.addEdge(2, 4);
        g.addEdge(3, 5);
        g.addEdge(4, 6);

        System.out.print("BFS 탐색 순서: ");
        g.bfs(1);
    }
}
```
**시간 복잡도**
- 정점 방문: O(V)
- 간선 확인: O(E)
- 전체 시간 복잡도: `𝑂(𝑉 + 𝐸)`