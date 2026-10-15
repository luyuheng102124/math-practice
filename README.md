# 口算练习 MVP

Vite + Vue3 + TS + Tailwind 在线口算练习静态网站。

## 功能
- 题型选择：加法 / 减法 / 加减混合
- 自定义数字范围（减法保证不出现负数）
- 回车提交答案，自动判断对错并显示正确答案
- 统计答题数、正确数、正确率
- 错题自动收录（localStorage 持久化），可切换「错题合集」专项练习，答对后自动移出

## 运行
npm install
npm run dev

## 构建
npm run build
