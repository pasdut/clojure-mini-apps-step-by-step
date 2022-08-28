This is the source of the site.

The site is create via the org mode file blog.org.

The md files are generated via ox-hugo.

And finally hugo creates the actual html pages.

The hugo site is initially created via hugo new site ./hugo -f yml

The Theme used is  hugo-PaperMod https://adityatelange.github.io/hugo-PaperMod/posts/papermod/
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
git submodule update --init --recursive # needed when you reclone your repo (submodules may not get cloned automatically)
