---
layout:     post
title:      "常用算法使用环境"
subtitle:   " \"算法训练\""
date:       2018-11-03 12:00:00
author:     "A-SHIN"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
    - 算法
---

> “Yeah It's on. ”

## 前言
刷题杂记
## 正文  
* 时间复杂度限定可利用：hashmap等空间换时间、二分法、滑动窗口法
* 连续子数组相关问题尝试用滑动窗口法或建立累加和数组方法
* DSF用stack，BSF用queue
* 寻路问题可用DSF，最优解可用DP
* 钱币找零问题使用贪心算法
* 背包问题使用动态规划法
* N皇后问题使用回溯法

一般DFS形式：
```
int DFS(int i, int j, int n , int k, vector<vector<int>>& grid, vector<vector<int>>& dp)
{
	if (dp[i][j] > 0)
	{
		return dp[i][j];
	}
	int left = grid[i][j], right = grid[i][j], up = grid[i][j], down = grid[i][j];
	if (i > k-1 && grid[i- k][j] > grid[i][j])
	{
		left += DFS(i - k, j, n, k, grid, dp);
	}
	if (i < n-k && grid[i+ k][j] > grid[i][j])
	{
		right += DFS(i + k, j, n, k, grid, dp);
	}
	if (j > k-1 && grid[i][j- k] > grid[i][j])
	{
		up += DFS(i, j - k, n, k, grid, dp);
	}
	if (j < n-k && grid[i][j+ k] > grid[i][j])
	{
		down += DFS(i, j + k, n, k, grid, dp);
	}
	dp[i][j] = max(max(left, right), max(up, down));
	return dp[i][j];
}
```

## 后记  
