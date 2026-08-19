# 💣 minesweeper-prd

I asked Claude to build Minesweeper. Three words in, working game out. So I wondered
where the rest came from — because "make Minesweeper" doesn't tell you that clicking
one empty square opens two hundred, or that the first click must never lose, or that
you win by clearing safe squares rather than flagging mines.

So I asked Claude to write a PRD that would be needed if none of that were known. 📄 **[Read it →](PRD.md)**

The useful part is the nine rules tagged **[COMMONLY OMITTED]** — the ones you'd
never think to write down, and couldn't guess without them.

Then I had it build the game again, to that spec. 🎮 **[Play it →](https://dp-lewis.github.io/minesweeper-prd/game.html)**

Side note: I never told it what a PRD should contain. How it knew that is
another thing I'd like to understand — but that's for another day.
