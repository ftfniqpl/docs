# pyinstall打包

## 添加静态文件及动态导入包

项目中如果有用到aiohttp，不能打包

pyi-makespec -w xxx.py

pyinstaller -D xxx.spec

pyinstaller -F server.py -n server   #这样就只生成一个二进制文件
```
# -*- mode: python -*-

block_cipher = None


a = Analysis(['server.py'],
             pathex=['/Users/tony/Downloads/www/rowsea'],
             binaries=[],
             datas=[('./app/hyy/static', 'app/hyy/static'), ('./app/hyy/templates', 'app/hyy/templates')],
             hiddenimports=['app', 'app.hyy', 'app.hyy.config', 'app.hyy.urls', 'app.hyy.alembic'],
             hookspath=[''],
             runtime_hooks=[],
             excludes=[],
             win_no_prefer_redirects=False,
             win_private_assemblies=False,
             cipher=block_cipher,
             noarchive=False)
pyz = PYZ(a.pure, a.zipped_data,
             cipher=block_cipher)
exe = EXE(pyz,
          a.scripts,
          [],
          exclude_binaries=True,
          name='server',
          debug=False,
          bootloader_ignore_signals=False,
          strip=False,
          upx=True,
          console=True )
coll = COLLECT(exe,
               a.binaries,
               a.zipfiles,
               a.datas,
               strip=False,
               upx=True,
               name='server')
```