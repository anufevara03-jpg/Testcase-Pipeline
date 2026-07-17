# 快速提交到github

当用户未指定提交信息和目标分支时，默认使用「快速提交」并提交到 main 分支。若用户指定了提交信息，则使用该信息。若用户指定了目标分支，则提交到该分支。

# 格式

gitpush [提交信息] [目标分支]

# 提交所有文件

依次执行下列命令
git add .
git commit -m "快速提交"
git push -u origin main # 提交到 main 分支
