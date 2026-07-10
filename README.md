# Nyx Bootcamp — Daily Practice

Daily practice repo for **Gaurav**, **Vishal**, **Ashish**, and **Nishant**.

DSA questions are picked daily (easy-to-medium) from the [Apna College 375 Qs DSA sheet by Shradha Didi & Aman Bhaiya](https://github.com/GFGSC-RTU/All-DSA-Sheets/blob/main/Apna%20college%20375%20Qs%20DSA%20sheet/DSA%20by%20Shradha%20Didi%20%26%20Aman%20Bhaiya.xlsx). Frontend/Backend/CS-Core theory and assignments follow the mentoring roadmap.

## Structure

```
Frontend/Theory/Day-N-<topic>/README.md        — today's frontend topic + resources
Frontend/Assignment/Day-N-<topic>/README.md     — today's frontend hands-on task
Backend/Theory/Day-N-<topic>/README.md          — today's backend topic + resources
Backend/Assignment/Day-N-<topic>/README.md      — today's backend hands-on task
CS-Core/Theory/Day-N-<topic>/README.md          — today's CS-core topic (both tracks)
CS-Core/Assignment/Day-N-<topic>/README.md      — today's CS-core reflection task (both tracks)
DSA/Question/Day-N-<topic>/README.md            — today's DSA question + link
DSA/Solution/Day-N-<topic>/SOLUTION.md          — official brute-force + optimal solution, published at day's end
```

`N` is the shared day number across all four categories — the same "Day N" you see in the daily WhatsApp update.

## How it works

1. Every morning, a new `Day-N-<topic>` folder appears in each of the four categories.
2. Fill in your `DAY<N>_<YourName>.md` placeholder (already created for you in the matching `Assignment/` or `DSA/Question/` folder), then open a **Pull Request**.
3. At day's end, `DSA/Solution/Day-N-<topic>/SOLUTION.md` is published with both a **brute-force** and an **optimal** solution, so you can compare your approach.

## How to submit (PR workflow)

```bash
# one-time setup
git clone https://github.com/visen-vin/nyx-bootcamp.git
cd nyx-bootcamp

# each day, e.g. for the DSA question
git checkout -b <yourname>-day-N
# fill in your placeholder inside the matching Day-N folder, e.g.:
#   DSA/Question/Day-N-<topic>/DAY<N>_<YourName>.md
#   Frontend/Assignment/Day-N-<topic>/DAY<N>_<YourName>.md
git add DSA/Question/Day-N-<topic>/DAY<N>_<YourName>.md
git commit -m "Day N: <YourName> submission"
git push origin <yourname>-day-N
# then open a Pull Request on GitHub from your branch into main
```

No need to be a collaborator — fork the repo if you don't have push access, and open the PR from your fork.
