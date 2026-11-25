"""Single-file fractional indexing module with OrderedBoard, trace, CLI, and tests.

Author: Pranay Hiwanj <pranay.hiwanj@gmail.com>
"""

from __future__ import annotations

import argparse
import sys
from dataclasses import dataclass

# Base-62 alphabet ordered in standard ASCII/lexicographical order:
# '0'-'9' < 'A'-'Z' < 'a'-'z'
ALPHABET = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
BASE = len(ALPHABET)
CHAR_TO_VAL = {ch: idx for idx, ch in enumerate(ALPHABET)}
MIN_CHAR = ALPHABET[0]
MAX_CHAR = ALPHABET[-1]
MID_CHAR = ALPHABET[BASE // 2]


def _validate_key(key: str) -> None:
    """Ensure a key is non-empty and uses only characters from ALPHABET."""
    if not key:
        raise ValueError("Key must be a non-empty string.")
    for ch in key:
        if ch not in CHAR_TO_VAL:
            raise ValueError(f"Invalid character {ch!r} in key {key!r}.")


def generate_key_between(before: str | None, after: str | None) -> str:
    """Generate a lexicographical string key strictly between `before` and `after`.

    - `before=None, after=None` -> starting key in the middle of the alphabet
    - `before=None, after=x`    -> key before `x` (prepend)
    - `before=x, after=None`    -> key after `x` (append)
    - `before=a, after=b`       -> key strictly between `a` and `b`
    """
    if before is not None:
        _validate_key(before)
    if after is not None:
        _validate_key(after)
    if before is not None and after is not None and before >= after:
        raise ValueError(f"`before` ({before!r}) must be less than `after` ({after!r}).")

    if before is None and after is None:
        return MID_CHAR

    if before is None:
        assert after is not None
        first_idx = CHAR_TO_VAL[after[0]]
        if first_idx > 1:
            return ALPHABET[first_idx // 2]
        if len(after) > 1:
            return after[0]
        return MIN_CHAR + MID_CHAR

    if after is None:
        last_idx = CHAR_TO_VAL[before[0]]
        if last_idx < BASE - 2:
            return ALPHABET[(last_idx + BASE) // 2]
        if len(before) > 1:
            return before[0] + MID_CHAR
        return MAX_CHAR + MID_CHAR

    prefix: list[str] = []
    i = 0
    while True:
        b_ch = before[i] if i < len(before) else MIN_CHAR
        a_ch = after[i] if i < len(after) else MAX_CHAR

        if b_ch == a_ch:
            prefix.append(b_ch)
            i += 1
            continue

        b_idx = CHAR_TO_VAL[b_ch]
        a_idx = CHAR_TO_VAL[a_ch]

        if a_idx - b_idx > 1:
            mid_idx = (b_idx + a_idx) // 2
            prefix.append(ALPHABET[mid_idx])
            return "".join(prefix)

        prefix.append(b_ch)
        i += 1


@dataclass
class Card:
    id: str
    title: str
    rank: str


class OrderedBoard:
    """Helper for managing an ordered list or Kanban column using rank keys."""

    def __init__(self) -> None:
        self._cards: dict[str, Card] = {}

    def ordered_cards(self) -> list[Card]:
        return sorted(self._cards.values(), key=lambda card: card.rank)

    def add_at(self, card_id: str, title: str, index: int | None = None) -> Card:
        """Insert a new card at `index` (or at the end if `index` is None)."""
        cards = self.ordered_cards()
        if index is None or index >= len(cards):
            before_key = cards[-1].rank if cards else None
            after_key = None
        elif index <= 0:
            before_key = None
            after_key = cards[0].rank if cards else None
        else:
            before_key = cards[index - 1].rank
            after_key = cards[index].rank

        rank = generate_key_between(before_key, after_key)
        card = Card(id=card_id, title=title, rank=rank)
        self._cards[card_id] = card
        return card

    def move_to(self, card_id: str, new_index: int) -> Card:
        """Move an existing card to `new_index` without re-ranking other cards."""
        if card_id not in self._cards:
            raise KeyError(f"Unknown card id: {card_id!r}")

        others = [c for c in self.ordered_cards() if c.id != card_id]
        if new_index <= 0:
            before_key = None
            after_key = others[0].rank if others else None
        elif new_index >= len(others):
            before_key = others[-1].rank if others else None
            after_key = None
        else:
            before_key = others[new_index - 1].rank
            after_key = others[new_index].rank

        card = self._cards[card_id]
        card.rank = generate_key_between(before_key, after_key)
        return card


def trace_repeated_insertions(steps: int = 10) -> None:
    """Trace inserting `steps` times right before `after_key` in the same gap."""
    before_key = "V"
    after_key = "W"
    print(f"Tracing {steps} insertions strictly between {before_key!r} and {after_key!r}:")

    current_after = after_key
    keys: list[str] = []
    for step in range(1, steps + 1):
        new_key = generate_key_between(before_key, current_after)
        keys.append(new_key)
        print(f"  Step {step:2d}: before={before_key!r:6s} after={current_after!r:6s} -> new={new_key!r}")
        current_after = new_key

    # Verify strict ascending order from `before_key` through all generated keys to `after_key`
    chain = [before_key, *reversed(keys), after_key]
    assert all(chain[i] < chain[i + 1] for i in range(len(chain) - 1))
    print("Order check passed: all keys are strictly sorted.")


def run_board_demo() -> None:
    """Demonstrate adding and reordering cards on a Kanban-style board."""
    board = OrderedBoard()
    board.add_at("c1", "Set up repo")
    board.add_at("c2", "Implement fractional keys")
    board.add_at("c3", "Ship release")

    print("\nInitial board:")
    for c in board.ordered_cards():
        print(f"  [{c.rank}] {c.title} ({c.id})")

    # Move "Ship release" (c3) to index 1 (between c1 and c2)
    board.move_to("c3", 1)
    print("\nAfter moving 'Ship release' to position 1:")
    for c in board.ordered_cards():
        print(f"  [{c.rank}] {c.title} ({c.id})")


def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(
        prog="fractional_indexing.py",
        description="Generate fractional index keys, run traces, or test OrderedBoard.",
    )
    parser.add_argument("--before", default=None, help="Lower-bound key")
    parser.add_argument("--after", default=None, help="Upper-bound key")
    parser.add_argument(
        "--trace",
        type=int,
        nargs="?",
        const=10,
        default=None,
        metavar="N",
        help="Trace N repeated insertions in the same gap (default: 10)",
    )
    parser.add_argument(
        "--board-demo",
        action="store_true",
        help="Run an OrderedBoard add/move walkthrough",
    )
    return parser


def main(argv: list[str] | None = None) -> int:
    parser = build_parser()
    args = parser.parse_args(argv)

    if args.trace is not None:
        trace_repeated_insertions(args.trace)
        return 0

    if args.board_demo:
        run_board_demo()
        return 0

    try:
        key = generate_key_between(args.before, args.after)
    except ValueError as exc:
        print(f"Error: {exc}", file=sys.stderr)
        return 1

    print(key)
    return 0


# --- Tests (run with `pytest fractional_indexing.py`) ---


def test_ordered_board_add_and_move():
    board = OrderedBoard()
    board.add_at("1", "First")
    board.add_at("2", "Second")
    board.add_at("3", "Third")

    board.move_to("3", 1)
    titles = [c.title for c in board.ordered_cards()]
    assert titles == ["First", "Third", "Second"]

    ranks = [c.rank for c in board.ordered_cards()]
    assert ranks == sorted(ranks)


def test_ten_repeated_insertions():
    left, right = "V", "W"
    for _ in range(10):
        mid = generate_key_between(left, right)
        assert left < mid < right
        right = mid


if __name__ == "__main__":
    raise SystemExit(main())
