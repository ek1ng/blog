---
title: DFS
date: 2021-08-09 00:00:00
tags: [leetcode,algorithm]
---
# 回溯算法

**声明：文章内容为作者阅读[labuladong的算法笔记](labuladong.gitee.io/)后的个人笔记分享**

## 框架

```java
result = []
def backtrack(路径, 选择列表):
    if 满足结束条件:
        result.add(路径)
        return
    
    for 选择 in 选择列表:
        做选择
        backtrack(路径, 选择列表)
        撤销选择
```

## Leetcode46.全排列

```java
class Solution {
    List<List<Integer>> res = new LinkedList<>();
    /*主函数，输入一组不重复的数字，返回全排列 */
    public List<List<Integer>> permute(int[] nums) {
        // 记录[路径]
        LinkedList<Integer> track = new LinkedList<>();
        backtrack(nums, track);
        return res;
    }

    //路径：记录在track中
    //选择列表：nums中不存在于track中的元素
    //结束条件：nums中的元素都在track中出现
    void backtrack(int[] nums, LinkedList<Integer> track) {
        //触发结束条件
        if(track.size() == nums.length) {
            res.add(new LinkedList(track));
            return;
        }

        for (int i = 0; i < nums.length; i++) {
            //排除不合法的选择
            if (track.contains(nums[i]))
                continue;
            //做选择
            track.add(nums[i]);
            //进入下一层决策树
            backtrack(nums, track);
            //取消选择
            track.removeLast();
        }
    }

}
```

## Leetcode51.八皇后

```c++
class Solution {
public:
   vector<vector<string>> res;

/* 输入棋盘边长 n，返回所有合法的放置 */
vector<vector<string>> solveNQueens(int n) {
    // '.' 表示空，'Q' 表示皇后，初始化空棋盘。
    vector<string> board(n, string(n, '.'));
    backtrack(board, 0);
    return res;
}

// 路径：board 中小于 row 的那些行都已经成功放置了皇后
// 选择列表：第 row 行的所有列都是放置皇后的选择
// 结束条件：row 超过 board 的最后一行
void backtrack(vector<string>& board, int row) {
    // 触发结束条件
    if (row == board.size()) {
        res.push_back(board);
        return;
    }
    
    int n = board[row].size();
    for (int col = 0; col < n; col++) {
        // 排除不合法选择
        if (!isValid(board, row, col)) 
            continue;
        // 做选择
        board[row][col] = 'Q';
        // 进入下一行决策
        backtrack(board, row + 1);
        // 撤销选择
        board[row][col] = '.';
    }
}
/* 是否可以在 board[row][col] 放置皇后？ */
bool isValid(vector<string>& board, int row, int col) {
    int n = board.size();
    // 检查列是否有皇后互相冲突
    for (int i = 0; i < n; i++) {
        if (board[i][col] == 'Q')
            return false;
    }
    // 检查右上方是否有皇后互相冲突
    for (int i = row - 1, j = col + 1; 
            i >= 0 && j < n; i--, j++) {
        if (board[i][j] == 'Q')
            return false;
    }
    // 检查左上方是否有皇后互相冲突
    for (int i = row - 1, j = col - 1;
            i >= 0 && j >= 0; i--, j--) {
        if (board[i][j] == 'Q')
            return false;
    }
    return true;
}
};
```

