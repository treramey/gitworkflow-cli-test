# =============================================================================
# Gitworkflow Aliases for Release Management
# =============================================================================
# Add to your ~/.gitconfig or project .git/config
#
# Based on rocketraman's aliases with additions for release automation
# https://gist.github.com/rocketraman/1fdc93feb30aa00f6f3a9d7d732102a9
#
# Integration branches (customize if needed):
#   master - stable releases
#   maint  - maintenance/hotfixes
#   next   - beta/stabilization
#   pu     - proposed updates (alpha)
# =============================================================================

[alias]
    # =========================================================================
    # LOG & VISUALIZATION (from rocketraman)
    # =========================================================================

    # Pretty log with graph
    lg = log --graph --abbrev-commit --decorate --format=format:'%C(yellow)%h%C(reset) %C(normal)%s%C(reset) %C(dim white)%an%C(reset) %C(dim blue)(%ar)%C(reset) %C(dim black)%d%C(reset)'

    # Same with --first-parent (great for integration branch history)
    lgp = log --graph --abbrev-commit --decorate --format=format:'%C(yellow)%h%C(reset) %C(normal)%s%C(reset) %C(dim white)%an%C(reset) %C(dim blue)(%ar)%C(reset) %C(dim black)%d%C(reset)' --first-parent

    # =========================================================================
    # TOPIC MANAGEMENT (from rocketraman)
    # =========================================================================

    # List all topic branches (matches: initials/description format)
    topics = "!f() { \
        git branch --sort=committerdate -r | \
        sed -e 's|remotes/||g' -e 's|origin/||g' | \
        grep -E \"^[a-z]{1,3}/.*\" | \
        cut -c3- | \
        grep -vE '^(archive|backup|wip)/' | \
        uniq | grep -v HEAD; \
    }; f"

    # Oldest ancestor between integration branch and topic
    oldest-ancestor = !bash -c 'diff --old-line-format= --new-line-format= <(git rev-list --first-parent \"${1:-master}\") <(git rev-list --first-parent \"${2:-HEAD}\") | head -1' -

    # Topic log/diff relative to master
    topiclg = !sh -c \"git lg $(git oldest-ancestor origin/$(echo $1 | sed -e 's|origin/||g') ${2:-master})..origin/$(echo $1 | sed -e 's|origin/||g')\" -
    topicdiff = !sh -c \"git diff $(git oldest-ancestor origin/$(echo $1 | sed -e 's|origin/||g') ${2:-master})..origin/$(echo $1 | sed -e 's|origin/||g')\" -
    topicstat = !sh -c \"git --no-pager diff --stat $(git oldest-ancestor origin/$(echo $1 | sed -e 's|origin/||g') ${2:-master})..origin/$(echo $1 | sed -e 's|origin/||g')\" -

    # View commits from an already-merged topic
    mergedtopiclg = !sh -c \"git lg $(git oldest-ancestor $1^2 ${2:-master})..$1^2\" -
    mergedrevs = !sh -c \"git lg $1^-\" -

    # =========================================================================
    # TOPIC STATUS (rocketraman's 'where' - extended)
    # =========================================================================
    # Legend: = master | * maint | + next | - pu | . none

    where = "!bash -c '\
        while read topic; do \
            contains=$(git branch -r --contains origin/$topic 2>/dev/null); \
            if grep -q origin/maint <<<\"$contains\"; then symbol=\"*\"; \
            elif grep -q origin/master <<<\"$contains\"; then symbol=\"=\"; \
            elif grep -q origin/next <<<\"$contains\"; then symbol=\"+\"; \
            elif grep -q origin/pu <<<\"$contains\"; then symbol=\"-\"; \
            else symbol=\".\"; fi; \
            echo \"$symbol $topic\"; \
            if git notes --ref=branchnote list origin/$topic &>/dev/null; then \
                echo \"  $(git notes --ref=branchnote show origin/$topic)\"; \
            fi; \
        done < <(git topics)'"

    # Shorter version without notes
    where-short = "!bash -c '\
        while read topic; do \
            contains=$(git branch -r --contains origin/$topic 2>/dev/null); \
            if grep -q origin/master <<<\"$contains\"; then echo \"= $topic\"; \
            elif grep -q origin/next <<<\"$contains\"; then echo \"+ $topic\"; \
            elif grep -q origin/pu <<<\"$contains\"; then echo \"- $topic\"; \
            else echo \". $topic\"; fi; \
        done < <(git topics)'"

    # Topics ready to graduate (in next but not master)
    ready-to-graduate = "!git log origin/master..origin/next --merges --first-parent --pretty=format:'%s' | grep -oE \"Merge branch '[^']+\" | sed \"s/Merge branch '//g\" | sort -u"

    # Topics not yet in any integration branch
    orphan-topics = "!bash -c '\
        while read topic; do \
            contains=$(git branch -r --contains origin/$topic 2>/dev/null); \
            if ! grep -qE \"origin/(master|maint|next|pu)\" <<<\"$contains\"; then \
                echo \"$topic\"; \
            fi; \
        done < <(git topics)'"

    # =========================================================================
    # RELEASE MANAGEMENT (new)
    # =========================================================================

    # Check if maint needs to be merged to master before release
    maint-status = "!f() { \
        commits=$(git log master..maint --oneline 2>/dev/null); \
        if [ -n \"$commits\" ]; then \
            echo \"⚠️  maint has commits not in master:\"; \
            echo \"$commits\"; \
            echo \"\"; \
            echo \"Run: git checkout master && git merge maint\"; \
        else \
            echo \"✅ maint is up to date with master\"; \
        fi; \
    }; f"

    # Pre-release checklist
    release-check = "!f() { \
        echo \"=== Release Checklist ===\"; \
        echo \"\"; \
        echo \"1. Maint status:\"; \
        git maint-status; \
        echo \"\"; \
        echo \"2. Topics ready to graduate:\"; \
        git ready-to-graduate || echo \"   (none)\"; \
        echo \"\"; \
        echo \"3. Current version:\"; \
        git describe --tags --abbrev=0 2>/dev/null || echo \"   (no tags)\"; \
        echo \"\"; \
        echo \"4. Commits since last tag:\"; \
        lasttag=$(git describe --tags --abbrev=0 2>/dev/null); \
        if [ -n \"$lasttag\" ]; then \
            git log $lasttag..HEAD --oneline | wc -l | xargs echo \"  \"; \
        else \
            echo \"   N/A\"; \
        fi; \
    }; f"

    # Show what would be in the next release
    release-preview = "!f() { \
        lasttag=$(git describe --tags --abbrev=0 2>/dev/null); \
        echo \"=== Changes since ${lasttag:-beginning} ===\"; \
        echo \"\"; \
        echo \"Features:\"; \
        git log ${lasttag:+$lasttag..}HEAD --pretty=format:'  * %s (%h)' --grep='^feat' | head -20; \
        echo \"\"; \
        echo \"Fixes:\"; \
        git log ${lasttag:+$lasttag..}HEAD --pretty=format:'  * %s (%h)' --grep='^fix' | head -20; \
        echo \"\"; \
        echo \"Merged topics:\"; \
        git log ${lasttag:+$lasttag..}HEAD --merges --first-parent --pretty=format:'%s' | \
            grep -oE \"Merge branch '[^']+\" | sed \"s/Merge branch '/  * /g\" | head -20; \
    }; f"

    # =========================================================================
    # INTEGRATION BRANCH REBUILDING (new)
    # =========================================================================

    # Rebuild pu from master (two-phase, sauerj style)
    rebuild-pu = "!f() { \
        echo \"Rebuilding pu...\"; \
        git checkout -B pu origin/master; \
        echo \"Phase 1: Topics from next\"; \
        for topic in $(git log origin/master..origin/next --merges --first-parent --pretty=format:'%s' | grep -oE \"Merge branch '[^']+\" | sed \"s/Merge branch '//g;s/'.*//g\" | sort -u); do \
            if git rev-parse --verify origin/$topic &>/dev/null; then \
                if ! git merge-base --is-ancestor origin/$topic origin/master 2>/dev/null; then \
                    echo \"  Merging: $topic\"; \
                    git merge --no-ff --no-edit origin/$topic -m \"Merge branch '$topic' into pu\" || git merge --abort; \
                fi; \
            fi; \
        done; \
        git commit --allow-empty -m \"### Match 'next'\" 2>/dev/null || true; \
        echo \"Phase 2: Proposed topics\"; \
        for topic in $(git topics); do \
            case $topic in wip/*) continue;; esac; \
            if git rev-parse --verify origin/$topic &>/dev/null; then \
                if ! git merge-base --is-ancestor origin/$topic HEAD 2>/dev/null; then \
                    if ! git merge-base --is-ancestor origin/$topic origin/master 2>/dev/null; then \
                        echo \"  Merging: $topic\"; \
                        git merge --no-ff --no-edit origin/$topic -m \"Merge branch '$topic' into pu\" || git merge --abort; \
                    fi; \
                fi; \
            fi; \
        done; \
        echo \"Done. Run 'git push origin pu --force-with-lease' to publish.\"; \
    }; f"

    # Rebuild next (preserve non-graduated topics)
    rebuild-next = "!f() { \
        echo \"Rebuilding next...\"; \
        topics=$(git log origin/master..origin/next --merges --first-parent --pretty=format:'%s' | grep -oE \"Merge branch '[^']+\" | sed \"s/Merge branch '//g;s/'.*//g\" | sort -u); \
        git checkout -B next origin/master; \
        for topic in $topics; do \
            if git rev-parse --verify origin/$topic &>/dev/null; then \
                if ! git merge-base --is-ancestor origin/$topic origin/master 2>/dev/null; then \
                    echo \"  Merging: $topic\"; \
                    git merge --no-ff --no-edit origin/$topic -m \"Merge branch '$topic' into next\" || git merge --abort; \
                fi; \
            fi; \
        done; \
        echo \"Done. Run 'git push origin next --force-with-lease' to publish.\"; \
    }; f"

    # =========================================================================
    # MERGING (from rocketraman)
    # =========================================================================

    m = merge --no-ff

    # Merge remote branch with clean message
    mb = "!f() { : git merge; \
        branch=\"$1\"; \
        if [[ $branch != origin/* ]]; then branch=\"origin/$branch\"; fi; \
        git merge --no-ff $branch -m \"Merge branch '$(echo $1 | sed -e 's|origin/||g')' into $(git symbolic-ref HEAD | sed -e 's|refs/heads/||g')\"; \
    }; f"

    # Graduate a topic to master (merge from next)
    graduate = "!f() { \
        topic=\"$1\"; \
        if [ -z \"$topic\" ]; then echo \"Usage: git graduate <topic>\"; exit 1; fi; \
        echo \"Graduating $topic to master...\"; \
        git checkout master; \
        git merge --no-ff origin/$topic -m \"Merge branch '$topic'\"; \
        echo \"Done. Don't forget to:\"; \
        echo \"  1. git push origin master\"; \
        echo \"  2. git rebuild-next\"; \
        echo \"  3. git rebuild-pu\"; \
    }; f"

    # =========================================================================
    # BRANCH NOTES (from rocketraman)
    # =========================================================================
    # Requires: git config --add remote.origin.push '+refs/notes/branchnote:refs/notes/branchnote'
    #           git config --add remote.origin.fetch '+refs/notes/branchnote:refs/notes/branchnote'

    branchnote = notes --ref=branchnote append
    branchnoterm = notes --ref=branchnote remove
    branchnoteshow = "!f() { git notes --ref=branchnote show origin/$1 2>/dev/null || echo '(no note)'; }; f"

    # =========================================================================
    # UTILITIES
    # =========================================================================

    # Branches not merged to a target
    tomerge = !sh -c 'git branch -r --no-merged ${2:-HEAD} | grep -Ev "HEAD" | grep -Ev \"(\\*|master|maint|next|pu)\" | grep ${1:-.}' -

    # List branches merged into a range (useful for release notes)
    mergedinto = "!f() { git lgp $1 | sed -e \"s|.*'\\(.*\\)'.*|\\1|g\" -e \"s|origin/||g\" | awk '!x[$0]++' | tac; }; f"

    # Fast-forward current branch from origin
    ff = !sh -c 'branch=$(git symbolic-ref HEAD | cut -d \"/\" -f 3-) && git merge --ff-only origin/$branch' -

    # Fetch all with prune
    fap = fetch --all -p -t

[rerere]
    # Remember conflict resolutions (essential for gitworkflow)
    enabled = true
