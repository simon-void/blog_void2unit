# void2unit

my dev blog, which you can read [here](https://void2unit.onrender.com)

## Hugo usage

- `hugo` to build the blog into the public-directory
- `hugo server` to [localhost:1313](http://localhost:1313/)
- `hugo new post/headline-with-hyphens.md` to initialize a new blog post in `./content/post/` using the post archetype
- update paper theme by 
```shell
cd themes/
rm -rf hugo-paper/
git git clone https://github.com/nanxiaobei/hugo-paper.git
rm -rf hugo-paper/.git
cd ..
```
