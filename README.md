name: Update Dynamic Profile

on:
  schedule:
    - cron: "0 */6 * * *"   # every 6 hours
  workflow_dispatch:

# FIX 1: Grant permission to write back to the repo
permissions:
  contents: write

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repo
        uses: actions/checkout@v3

      - name: Fetch recent activity
        run: |
          echo "" > activity.md
          # FIX 2: Added Authorization header to prevent rate limiting
          RECENT=$(curl -s -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" "https://api.github.com/users/raymond-01/events?per_page=10" | jq '.[] | select(.type=="PushEvent")')
          
          if [ -z "$RECENT" ]; then
            echo "No recent activity found." > activity.md
          else
            echo "### 🔥 Latest Activity" >> activity.md
            echo "" >> activity.md
            # Note: Timestamps will be raw (e.g., 2023-10-25T14:00:00Z). Formatting them in bash is hard; this is acceptable.
            echo "$RECENT" | jq -r '"- **\(.repo.name)** — pushed on \(.created_at)"' >> activity.md
          fi

      - name: Fetch latest repos
        run: |
          # FIX 2: Added Authorization header here too
          curl -s -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" "https://api.github.com/users/raymond-01/repos?sort=updated&per_page=5" \
          | jq -r '.[] | "- [\(.name)](\(.html_url)) — updated \(.updated_at)"' > repos.md

      - name: Inject into README
        run: |
          awk '
            BEGIN {activity=0; projects=0}
            // {print; while ((getline line < "activity.md") > 0) print line; activity=1; next}
            // {activity=0}
            // {print; while ((getline line < "repos.md") > 0) print line; projects=1; next}
            // {projects=0}
            activity==0 && projects==0 {print}
          ' README.md > NEWREADME.md
          mv NEWREADME.md README.md

      - name: Commit and push changes
        run: |
          git config user.name "GitHub Action"
          git config user.email "action@github.com"
          git add README.md
          # This prevents the workflow from failing if there are no new updates to commit
          git commit -m "Auto-update profile" || echo "No changes to commit"
          git push
