# Mentor Squad — Daily DSA Practice

Daily DSA practice repo for **Gaurav**, **Vishal**, **Ashish**, and **Nishant**.

Questions are picked daily (easy-to-medium) from the [Apna College 375 Qs DSA sheet by Shradha Didi & Aman Bhaiya](https://github.com/GFGSC-RTU/All-DSA-Sheets/blob/main/Apna%20college%20375%20Qs%20DSA%20sheet/DSA%20by%20Shradha%20Didi%20%26%20Aman%20Bhaiya.xlsx).

## How it works

1. Every day a new `Day-N/` folder appears with `README.md` — the day's question, topic, and a link to solve it on GeeksforGeeks/LeetCode.
2. Solve it, then open a **Pull Request** adding your own file inside that day's folder, named after yourself (e.g. `Day-N/Gaurav.md` or `Day-N/Gaurav.py`) with your solution attempt.
3. At day's end, `Day-N/SOLUTION.md` is published with both a **brute-force** and an **optimal** solution, so you can compare your approach.

## How to submit your solution (PR workflow)

```bash
# one-time setup
git clone https://github.com/visen-vin/mentor-squad-dsa.git
cd mentor-squad-dsa

# each day
git checkout -b <yourname>-day-N
# add your file: Day-N/<YourName>.md (or .py/.java/.js — any language is fine)
git add Day-N/<YourName>.md
git commit -m "Day N: <YourName> solution"
git push origin <yourname>-day-N
# then open a Pull Request on GitHub from your branch into main
```

No need to be a collaborator — fork the repo if you don't have push access, and open the PR from your fork.
