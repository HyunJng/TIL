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

---

# DFS와 BFS의 성능 비교

격자 탐색 문제를 해결할 때 자주 사용되는 기법은 **DFS(깊이 우선 탐색)** 와 **BFS(너비 우선 탐색)** 이다. 
두 방법은 이론적으로는 동일한 시간 복잡도를 가지지만, 실제 실행 속도에서는 차이가 발생한다.


## 1. BFS가 느려질 수 있는 원인

1. **큐 자료구조 오버헤드**
   BFS는 `Queue.offer()`와 `Queue.poll()` 연산을 반복적으로 수행해야 한다. 특히 Java에서 `LinkedList` 기반 큐는 내부적으로 노드 객체 생성과 링크 관리가 필요하여 실행 시간이 늘어날 수 있다.

2. **객체 생성 부담**
   BFS에서는 탐색 과정에서 대상을 큐에 저장한다. 이때 새로운 배열이나 객체가 지속적으로 생성되어 가비지 컬렉션(GC) 부담으로 이어질 수 있다.

3. **메모리 접근 패턴**
   BFS는 탐색이 격자 전체로 넓게 확산되므로, 메모리 접근이 불규칙하게 발생한다. 이로 인해 캐시 적중률이 낮아지고 성능 저하가 발생할 수 있다.

## 2. DFS가 상대적으로 빠른 이유

1. **스택 기반 탐색**
   DFS는 함수 호출 스택(재귀)이나 단순 배열 스택을 통해 구현할 수 있다. 큐 자료구조에 비해 관리 비용이 적다.

2. **연속적인 메모리 접근**
   DFS는 한 방향으로 깊게 탐색을 이어가기 때문에 인접한 영역을 연속적으로 접근하는 경우가 많다. 이는 캐시 효율을 높여 성능상 이점을 준다.

3. **구현 단순성**
   코드가 간단하며 불필요한 자료구조 사용을 줄일 수 있어 실행 시간이 단축되는 경향이 있다.

## 3. 주의사항

* DFS를 재귀로 구현할 경우 입력 크기가 커질 때 스택 오버플로우 위험이 존재한다.
* 예를 들어 `N = 1000` 이상인 경우 재귀 호출 깊이가 수천 단위에 도달할 수 있으며, 이때는 BFS가 더 안정적이다.

## 결론

* **시간 복잡도**: DFS와 BFS 모두 O(N²)로 동일하다.
* **실행 시간**: 일반적으로 DFS가 BFS보다 빠른 경향이 있다. 이는 자료구조 오버헤드, 객체 생성 비용, 캐시 활용도 차이 때문이다.
* **안정성**: 입력 크기가 크면 DFS는 재귀 한계로 인해 위험할 수 있으며, 이 경우 BFS가 더 적합하다.