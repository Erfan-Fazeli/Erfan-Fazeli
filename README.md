# file: README.md

<p align="center">
  <img src="./terminal_whoami.svg" alt="Terminal whoami card for Erfan Fazeli" />
</p>

---

# file: requirements.txt

requests==2.32.3

---

# file: generate_svg.py

from __future__ import annotations

import datetime as dt
import os
from dataclasses import dataclass

import requests


GITHUB_GRAPHQL_URL = "https://api.github.com/graphql"
USERNAME = "Erfan-Fazeli"
DISPLAY_NAME = "Erfan Fazeli"
ROLE = "DevOps & Programmer"
LOCATION = "Iran"
ACCENT = "#58A6FF"
OUTPUT_FILE = "terminal_whoami.svg"

ABOUT_LINES = [
    "Specialized in DevOps engineering with deep expertise in Linux, Docker, and cloud infrastructure.",
    "Network engineering knowledge complementing my DevOps journey.",
    "Passionate about IoT & home automation with Arduino, Raspberry Pi, and ESP32.",
    "Electronics & Automation Systems Engineer.",
    "Familiar with Scrum methodology and product development lifecycle.",
    "Growing experience in Backend Development with focus on scalable solutions.",
]

STACK_LINES = [
    "Infrastructure & Containerization",
    "Network Engineering",
    "IoT Development",
    "Automation Solutions",
    "Team Leadership & Agile Project Management",
]


@dataclass
class Stats:
    username: str
    name: str
    followers: int
    following: int
    repositories: int
    stars_received: int
    commits_last_year: int


def graphql_request(query: str, variables: dict) -> dict:
    token = os.environ["GH_TOKEN"]
    response = requests.post(
        GITHUB_GRAPHQL_URL,
        json={"query": query, "variables": variables},
        headers={
            "Authorization": f"Bearer {token}",
            "Content-Type": "application/json",
        },
        timeout=30,
    )
    response.raise_for_status()
    payload = response.json()
    if "errors" in payload:
        raise RuntimeError(payload["errors"])
    return payload["data"]


def fetch_stats(username: str) -> Stats:
    query = """
    query($login: String!) {
      user(login: $login) {
        login
        name
        followers {
          totalCount
        }
        following {
          totalCount
        }
        repositories(ownerAffiliations: OWNER, isFork: false) {
          totalCount
        }
        contributionsCollection {
          contributionCalendar {
            totalContributions
          }
        }
        repositoriesContributedTo(contributionTypes: [COMMIT], includeUserRepositories: true) {
          totalCount
        }
      }
      userStarred: user(login: $login) {
        repositories(first: 100, ownerAffiliations: OWNER, isFork: false, privacy: PUBLIC, orderBy: {field: UPDATED_AT, direction: DESC}) {
          nodes {
            stargazerCount
          }
        }
      }
    }
    """
    data = graphql_request(query, {"login": username})
    user = data["user"]
    repos = data["userStarred"]["repositories"]["nodes"] or []
    stars_received = sum(repo["stargazerCount"] for repo in repos)

    return Stats(
        username=user["login"],
        name=user["name"] or username,
        followers=user["followers"]["totalCount"],
        following=user["following"]["totalCount"],
        repositories=user["repositories"]["totalCount"],
        stars_received=stars_received,
        commits_last_year=user["contributionsCollection"]["contributionCalendar"]["totalContributions"],
    )


def xml_escape(value: str) -> str:
    return (
        str(value)
        .replace("&", "&amp;")
        .replace("<", "&lt;")
        .replace(">", "&gt;")
        .replace('"', "&quot;")
        .replace("'", "&apos;")
    )


def build_lines(stats: Stats) -> list[tuple[str, str]]:
    lines: list[tuple[str, str]] = [
        ("prompt", "$ whoami"),
        ("space", ""),
        ("output", f"name: {DISPLAY_NAME}"),
        ("output", f"username: @{stats.username}"),
        ("output", f"role: {ROLE}"),
        ("output", f"location: {LOCATION}"),
        ("space", ""),
        ("prompt", "$ about"),
    ]

    lines.extend(("output", line) for line in ABOUT_LINES)
    lines.append(("space", ""))
    lines.append(("prompt", "$ stack"))
    lines.extend(("output", f"- {line}") for line in STACK_LINES)
    lines.append(("space", ""))
    lines.append(("prompt", "$ stats"))
    lines.extend(
        [
            ("output", f"repositories: {stats.repositories}"),
            ("output", f"followers: {stats.followers}"),
            ("output", f"following: {stats.following}"),
            ("output", f"stars_received: {stats.stars_received}"),
            ("output", f"commits_last_365_days: {stats.commits_last_year}"),
        ]
    )
    lines.append(("space", ""))
    lines.append(("prompt", "$ echo 'welcome to my profile'"))
    lines.append(("output", "welcome to my profile"))
    lines.append(("prompt", "$ _"))
    return lines


def build_svg(stats: Stats) -> str:
    lines = build_lines(stats)
    width = 980
    height = 760
    line_height = 24
    start_y = 95

    text_nodes: list[str] = []
    visible_index = 0

    for kind, content in lines:
        if kind == "space":
            visible_index += 1
            continue

        y = start_y + visible_index * line_height
        safe_text = xml_escape(content)
        css_class = "prompt" if kind == "prompt" else "output"
        delay = visible_index * 0.28
        text_nodes.append(
            f'<text x="34" y="{y}" class="{css_class}" style="animation-delay:{delay:.2f}s">{safe_text}</text>'
        )
        visible_index += 1

    generated_at = dt.datetime.utcnow().strftime("%Y-%m-%d %H:%M UTC")

    return f"""<svg width="{width}" height="{height}" viewBox="0 0 {width} {height}" xmlns="http://www.w3.org/2000/svg" role="img" aria-labelledby="title desc">
  <title id="title">Terminal whoami card for {xml_escape(stats.username)}</title>
  <desc id="desc">Animated terminal style GitHub profile card with whoami, about, stack, and stats.</desc>

  <style>
    .window {{
      fill: #0d1117;
      stroke: #30363d;
      stroke-width: 1;
    }}

    .topbar {{
      fill: #161b22;
    }}

    .dot-red {{ fill: #ff5f56; }}
    .dot-yellow {{ fill: #ffbd2e; }}
    .dot-green {{ fill: #27c93f; }}

    .title {{
      font: 600 14px ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
      fill: #8b949e;
    }}

    .prompt, .output {{
      font: 500 18px ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
      opacity: 0;
      animation: reveal 0.001s linear forwards;
    }}

    .prompt {{
      fill: {ACCENT};
    }}

    .output {{
      fill: #c9d1d9;
    }}

    .footer {{
      font: 400 12px ui-monospace, SFMono-Regular, Menlo, Consolas, monospace;
      fill: #6e7681;
    }}

    @keyframes reveal {{
      to {{
        opacity: 1;
      }}
    }}
  </style>

  <rect x="10" y="10" width="{width - 20}" height="{height - 20}" rx="18" class="window" />
  <rect x="10" y="10" width="{width - 20}" height="46" rx="18" class="topbar" />
  <rect x="10" y="38" width="{width - 20}" height="18" fill="#161b22" />

  <circle cx="34" cy="33" r="6.5" class="dot-red" />
  <circle cx="56" cy="33" r="6.5" class="dot-yellow" />
  <circle cx="78" cy="33" r="6.5" class="dot-green" />

  <text x="110" y="37" class="title">erfan@github: ~/profile</text>

  {''.join(text_nodes)}

  <text x="34" y="{height - 22}" class="footer">generated automatically • {generated_at}</text>
</svg>"""


def main() -> None:
    stats = fetch_stats(USERNAME)
    svg = build_svg(stats)
    with open(OUTPUT_FILE, "w", encoding="utf-8") as file:
        file.write(svg)


if __name__ == "__main__":
    main()

---

# file: .github/workflows/update-terminal-svg.yml

name: Update terminal profile SVG

on:
  workflow_dispatch:
  schedule:
    - cron: "0 */12 * * *"

permissions:
  contents: write

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Generate SVG
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: python generate_svg.py

      - name: Commit generated SVG
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add terminal_whoami.svg
          git diff --cached --quiet || git commit -m "Update terminal SVG"
          git push
