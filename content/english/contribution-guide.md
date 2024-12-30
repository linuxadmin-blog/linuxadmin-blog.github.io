---
title: "Contributing"
layout: "contribution-guide"
draft: false
---

We invite you to join our community by sharing your expertise, collaborating on projects, and engaging with fellow professionals. For a respectful, harassment-free environment, you can view the [Contribution Guide](https://linuxadmin.blog/contribution-guide) for further information.

Please read our [Code of Conduct](https://linuxadmin.blog/code-of-conduct), which outlines our guidelines and expectations for all members to foster an inclusive and welcoming space.

You may follow steps below to create a profile:

1. Install latest versions of Go, Hugo, NodeJS.
2. Clone the repository and start Hugo server:
```sh
git clone https://github.com/linuxadmin-blog/linuxadmin-blog.github.io.git ~/linuxadmin-blog
cd ~/linuxadmin-blog && cp themes/bookworm-light-hugo/package.json ~/linuxadmin.blog/package.json && npm install
npm run dev
# this will start a web server at http://localhost:1313
```
3. Create an author profile in `/content/english/author/<name-surname>.md` with author template:
```Markdown
---
title: "Name Surname"
image: "images/author/<name-surname.png/svg/jpg>"
email: "<email-address>"
date: <create-date>
draft: false
social:
- icon: "la-github"
  link: "https://github.com/<username>"
- icon: "la-linkedin-in"
  link: "https://linkedin.com/in/<username>"
---

Introductory paragraph...
```
4. Go to the author page `http://localhost:1313/author`.
5. Push changes to a branch which is named as `create-author-<name-surname>`.
6. Wait for [Github Action](https://github.com/linuxadmin-blog/linuxadmin-blog.github.io/actions) to deploy the changes.
7. Now you are a new member of our community :)