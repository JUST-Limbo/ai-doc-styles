# 基于 Webpack 5 的 Vue 2 SSR 工程实践综述

## 前言

技术文章，尤其是前端技术文章具有时效性。

若文中出现 breaking change、事实错误或表述不当，欢迎在评论区或仓库 issue 中指出。

相关仓库：

- [前端示例仓库](https://github.com/JUST-Limbo/vue2-ssr-practice)
- [接口示例仓库](https://github.com/JUST-Limbo/mock-backend)
- [原文档所在仓库](https://github.com/JUST-Limbo/informal-essay)

阅读本文前建议先通读官方文档：[Vue 2 服务器端渲染指南](https://v2.ssr.vuejs.org/zh/)。

若你的目标仅是调研 Vue SSR 技术选型，社区侧更常见、维护成本更低的做法是直接采用 Nuxt 3 等一体化方案。本文聚焦「手写 Webpack 5 配置」这一偏工程化视角。

## 摘要

技术社区里围绕 Vue 2 与 SSR 的文章，多数基于 `vue/cli@3` 或 `vue/cli@4` 搭建。

[示例仓库](https://github.com/JUST-Limbo/vue2-ssr-practice)在 [vue-hackernews-2.0](https://github.com/vuejs/vue-hackernews-2.0) 思路上脱离 Vue CLI，将 Webpack 3 升级到 Webpack 5，顺带处理一批老旧依赖（例如 `node-sass`），并同时给出 CSR 与 SSR 两套构建配置。

下文按模块拆解示例仓库中的 Webpack 配置，重点说明 SSR 开发模式下的组织方式与常见坑位。

## 环境与依赖版本

| 名称 | 版本 | 说明 |
| --- | --- | --- |
| nvm | 1.1.9 | Node 版本管理 |
| Node.js | 16.20.0 | Webpack 5 最低支持 Node 10.13；pnpm 最低支持 Node 16.14；此处取 16.20.0 |
| pnpm | 8.6.1 | 包管理器 |
| vue | 2.6.x | 版本需与 `vue-loader`、`vue-template-compiler`、`vue-server-renderer` 对齐；Vue 2.7 起内置组合式 API，示例为稳妥起见固定在 2.6 末版（写作时为 2.6.14） |
| webpack | 5.85.0 | 相较 Webpack 3、4，构建性能与文档可读性更好 |
| express | 4.x | 开发期本地服务端示例 |
| chalk | 4.x | chalk 5 为 ESM 形态，此处示例用 4 以便 `require` |

## 从空白目录初始化 Webpack 工程

```shell
pnpm init
pnpm i webpack@5 webpack-cli@5 -D
npx webpack init
```

在空白目录中依次执行上述命令，可得到：

1. 已初始化的 `package.json` 与基础目录结构
2. 一份极简的 Webpack 配置骨架
3. 一组常用样式链依赖（`sass`、`sass-loader`、`style-loader`、`postcss`、`postcss-loader` 等），减少手工安装步骤

## 路由与 history 模式

先说结论：在`SSR`的场景里，`vue-router`只能采用`history`模式。

**为什么不能使用`hash`？**

因为在`hash`模式下，页面URL的hash内容并不会随着请求一起发送到服务器中，而`history`模式没有这个问题。

也就是说：

当你试图访问`http://localhost:9500/#/user/1`时，可以在F12的network中观察到，浏览器实际上访问的是`http://localhost:9500/`。

而访问`http://localhost:9500/user/1`时，浏览器实际上访问的是`http://localhost:9500/user/1`，这是符合`SSR`期望的。

**`history`模式有哪些需要注意的？**

如果你仅仅是配置了`mode: 'history'`，没有进行其他配置，那么你将会遭遇：

1. `CSR`应用开发场景下**通过命令式导航跳转**的方式能进入`http://localhost:9800/xxx`
2. `CSR`应用开发场景下**直接输入**链接访问`http://localhost:9800/xxx`，页面返回`Cannot GET /xxx`
3. 在`nginx`部署生产完成后直接访问`http://xxx.com/xxx`会返回404

上述问题本质上是同一个问题：当访问`http://localhost:9800/xxx`时，浏览器会尝试访问`http://localhost:9800`静态服务器中`/xxx`目录下的`index.html`。

而观察过构建目录`dist`的开发者都知道，不做特殊处理的情况下`dist`中不会构建出`/xxx`，这也是为什么返回404的原因，即：`/xxx/index.html`根本不存在。

本文只介绍`webpack`解决该问题的办法（注意这是针对`CSR`应用开发环境），至于`nginx`的解决方案建议咨询所在公司运维人员。

解决思路是：将所有请求转发到`http://localhost:9800/index.html`

```js
devServer: {
    historyApiFallback: {
        rewrites: [
            {
                from: /\//,
                to: '/index.html',
            },
        ],
    },
    // 或
    // historyApiFallback:true,
},
```

对于`devServer.historyApiFallback`配置，更多了解请点击[DevServer | webpack 中文文档 (docschina.org)](https://webpack.docschina.org/configuration/dev-server/#devserverhistoryapifallback)

**本文聊的是`SSR`，为什么要花篇幅描述`CSR`下的`history`处理呢？**

因为`SSR`的容灾降级需要用到`CSR`。

## 公共配置：webpack.base.config.js

在`SSR`模式下，关于样式文件的`loader`应采用`vue-style-loader`。它和`style-loader`的不同在于：它支持`vue ssr`。（via [vuejs/vue-style-loader: 💅 vue style loader module for webpack (github.com)](https://github.com/vuejs/vue-style-loader#differences-from-style-loader) ）

```js
const path = require("path")
const MiniCssExtractPlugin = require("mini-css-extract-plugin")
const { VueLoaderPlugin } = require("vue-loader")
const CopyPlugin = require("copy-webpack-plugin")
const CssMinimizerPlugin = require("css-minimizer-webpack-plugin")
const TerserPlugin = require("terser-webpack-plugin")

const NODE_ENV = process.env.NODE_ENV
const isProd = NODE_ENV == "production"

const modeMap = {
	production: "production",
	development: "development"
}

const stylesHandler = isProd ? MiniCssExtractPlugin.loader : "vue-style-loader"

const OptimizationMap = {
	development: {
		chunkIds: "named",
		moduleIds: "named",
		usedExports: false, // 树摇
		splitChunks: {
			chunks: "all",
			minSize: 1024 * 20,
			maxSize: 1024 * 500,
			minChunks: 2
		},
		runtimeChunk: {
			name: "runtime"
		}
	},
	production: {
		chunkIds: "deterministic",
		moduleIds: "deterministic",
		usedExports: true, // 树摇
		splitChunks: {
			chunks: "all",
			minSize: 1024 * 20,
			// maxSize: 1024 * 244, // 拆包会导致包数激增,不拆的话可能会出现单包过大的问题
			minChunks: 2, // 分包不能太细 不合并导致并发请求过多 浏览器有并发请求数量限制
			cacheGroups: {
				commons: {
					test: /[\\/]node_modules[\\/]/,
					name: "vendors",
					priority: 1
				},
				manifest: {
					name: "manifest"
				}
			}
		},
		// 将 optimization.runtimeChunk 设置为 true 或 'multiple'，会为每个入口添加一个只含有 runtime 的额外 chunk。
		runtimeChunk: {
			name: "runtime"
		},
		minimize: true,
		minimizer: [
			new TerserPlugin({
				parallel: true, // 多线程
				extractComments: false, // 注释单独提取
				terserOptions: {
					compress: {
						drop_console: true // 清除console输出
					},
					format: {
						comments: false // 清除注释
					},
					toplevel: true, //  声明提前
					keep_classnames: true // 类名不变
				}
			}),
			new CssMinimizerPlugin({
				minimizerOptions: {
					preset: [
						"default",
						{
							discardComments: { removeAll: true }
						}
					]
				}
			})
		]
	}
}

const config = {
	mode: modeMap[NODE_ENV] || "development",
	stats: "errors-only",
	infrastructureLogging: {
		level: "error"
	},
	output: {
		path: path.resolve(__dirname, "../dist"),
		publicPath: "/dist/",
		filename: isProd ? "js/[name].[contenthash:6].bundle.js" : "js/[name].bundle.js",
		chunkFilename: isProd ? "js/chunk_[name]_[contenthash:6].js" : "js/chunk_[name].js"
	},
	cache: !isProd,
	resolve: {
		extensions: [".js", ".vue", ".json", ".ts", ".html"],
		alias: {
			"@": path.resolve(__dirname, "../src"),
			vue$: "vue/dist/vue.esm.js"
		}
	},
	optimization: OptimizationMap[NODE_ENV],
	plugins: [
		new VueLoaderPlugin(),
		new CopyPlugin({
			patterns: [{ from: path.resolve(__dirname, "../public"), to: "public" }]
		})
	],
	module: {
		rules: [
			// vue-loader必须要在最外层,不能放入oneOf
			// https://github.com/vuejs/vue-loader/issues/1204#issuecomment-375739662
			// Note the rule for vue-loader must be at the top level
			{
				test: /\.vue$/i,
				include: [path.resolve(__dirname, "../src")],
				exclude: /node_modules/,
				use: ["vue-loader"]
			},
			{
				oneOf: [
					{
						test: /\.(js|jsx)$/i,
						exclude: /node_modules/,
						use: ["thread-loader", "babel-loader"]
					},
					{
						test: /\.s[ac]ss$/i,
						use: [
							stylesHandler,
							{
								loader: "css-loader",
								options: {
									importLoaders: 2,
									modules: {
										mode: "icss"
									}
								}
							},
							"postcss-loader",
							"sass-loader"
						]
					},
					{
						test: /\.css$/i,
						include: [/element-ui/],
						use: [
							MiniCssExtractPlugin.loader,
							{
								loader: "css-loader",
								options: {
									importLoaders: 1
								}
							},
							"postcss-loader"
						]
					},
					{
						test: /\.css$/i,
						exclude: [/element-ui/],
						use: [
							stylesHandler,
							{
								loader: "css-loader",
								options: {
									importLoaders: 1
								}
							},
							"postcss-loader"
						]
					},
					{
						test: /\.(eot|svg|ttf|woff|woff2|png|jpg|gif)$/i,
						type: "asset",
						parser: {
							dataUrlCondition: {
								maxSize: 10000
							}
						}
					}
				]
			}
		]
	}
}

module.exports = config

```

## 客户端配置：webpack.client.config.js

`SSR`模式不需要使用`html-webpack-plugin`。

不同的`WebpackBar`实例在初始化时要指定不同的`name`属性，否则在终端中构建进度条会重叠到一起。

```js
const { merge } = require("webpack-merge")
const base = require("./webpack.base.config")
const VueSSRClientPlugin = require("vue-server-renderer/client-plugin")
const MiniCssExtractPlugin = require("mini-css-extract-plugin")
const Dotenv = require("dotenv-webpack")
const CompressionPlugin = require("compression-webpack-plugin")
const WebpackBar = require("webpackbar")

const NODE_ENV = process.env.NODE_ENV
const isProd = NODE_ENV == "production"

const clientConfig = {
	entry: {
		app: "./src/entry-client.js"
	},
	resolve: {
		alias: {
			axiosInstance: "@/utils/request-client.js"
		}
	},
	plugins: [
		new VueSSRClientPlugin(),
		new WebpackBar({
			name: "Client",
			color: "#7ED321"
		})
	]
}

module.exports = (env, args) => {
	const APP_ENV = env.APP_ENV || "development"
	clientConfig.plugins.push(
		new Dotenv({
			path: `./envVariable/client/.env.${APP_ENV}`
		})
	)
	if (isProd) {
		clientConfig.devtool = false
		clientConfig.plugins.push(
			new MiniCssExtractPlugin({
				filename: "styles/[name].[contenthash:6].css"
			}),
			new CompressionPlugin({
				test: /\.(css|js)$/,
				minRatio: 0.7
			})
		)
	} else {
		clientConfig.plugins.push(
			new MiniCssExtractPlugin({
				filename: "styles/[name].css"
			})
		)
		clientConfig.devtool = "cheap-module-source-map"
	}
	return merge(base, clientConfig)
}

```

## 服务端配置：webpack.server.config.js

```js
const { merge } = require("webpack-merge")
const base = require("./webpack.base.config")
const nodeExternals = require("webpack-node-externals")
const MiniCssExtractPlugin = require("mini-css-extract-plugin")
const VueSSRServerPlugin = require("vue-server-renderer/server-plugin")
const Dotenv = require("dotenv-webpack")
const CompressionPlugin = require("compression-webpack-plugin")
const WebpackBar = require("webpackbar")
const FriendlyErrorsWebpackPlugin = require("friendly-errors-webpack-plugin")
const chalk = require("chalk")
const { defaultPort } = require("./setting")
const address = require("address")

// Error: Server-side bundle should have one single entry file. Avoid using CommonsChunkPlugin in the server config.
delete base.optimization

const NODE_ENV = process.env.NODE_ENV
const isProd = NODE_ENV == "production"
const serverConfig = {
	target: "node",
	devtool: "cheap-module-source-map",
	entry: "./src/entry-server.js",
	output: {
		filename: "server-bundle.js",
		libraryTarget: "commonjs2",
		// clean: true // 在生成文件之前清空 output 目录
	},
	optimization: {
		splitChunks: false
	},
	externals: nodeExternals({
		allowlist: [/\.css$/]
	}),
	resolve: {
		alias: {
			axiosInstance: "@/utils/request-server.js"
		}
	},
	plugins: [
		new VueSSRServerPlugin(),
		new WebpackBar({
			name: "Server",
			color: "#F5A623"
		})
	]
}

module.exports = (env, args) => {
	const APP_ENV = env.APP_ENV || "development"
	serverConfig.plugins.push(
		new Dotenv({
			path: `./envVariable/server/.env.${APP_ENV}`
		})
	)
	if (isProd) {
		serverConfig.plugins.push(
			new MiniCssExtractPlugin({
				filename: "styles/[name].[contenthash:6].css"
			}),
			new CompressionPlugin({
				test: /\.(css|js)$/,
				minRatio: 0.7
			})
		)
	} else {
		const port = args.port || defaultPort
		const LOCAL_IP = address.ip()
		serverConfig.plugins.push(
            new MiniCssExtractPlugin({
				filename: "styles/[name].css"
			}),
			new FriendlyErrorsWebpackPlugin({
				compilationSuccessInfo: {
					messages: [`  App running at:`, `  - Local:   ` + chalk.cyan(`http://localhost:${port}`), `  - Network: ` + chalk.cyan(`http://${LOCAL_IP}:${port}`)]
				},
				clearConsole: true
			})
		)
	}
	return merge(base, serverConfig)
}

```

## SPA 构建：webpack.spa.config.js

```js
const path = require('path');
const HtmlWebpackPlugin = require('html-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const WebpackBar = require('webpackbar');
const portfinder = require('portfinder');
const FriendlyErrorsWebpackPlugin = require('friendly-errors-webpack-plugin');
const address = require('address');
const chalk = require('chalk');
const { VueLoaderPlugin } = require('vue-loader');
const Dotenv = require('dotenv-webpack');
const CopyPlugin = require('copy-webpack-plugin');
const CssMinimizerPlugin = require('css-minimizer-webpack-plugin');
const TerserPlugin = require('terser-webpack-plugin');
const CompressionPlugin = require('compression-webpack-plugin');

const NODE_ENV = process.env.NODE_ENV;
const isProd = NODE_ENV == 'production';

const modeMap = {
    production: 'production',
    development: 'development',
};

const stylesHandler = isProd ? MiniCssExtractPlugin.loader : 'vue-style-loader';

const OptimizationMap = {
    development: {
        chunkIds: 'named',
        moduleIds: 'named',
        usedExports: false, // 树摇
        splitChunks: {
            chunks: 'all',
            minSize: 1024 * 20,
            maxSize: 1024 * 500,
            minChunks: 2,
        },
        runtimeChunk: {
            name: 'runtime',
        },
    },
    production: {
        chunkIds: 'deterministic',
        moduleIds: 'deterministic',
        usedExports: true, // 树摇
        splitChunks: {
            chunks: 'all',
            minSize: 1024 * 20,
            // maxSize: 1024 * 244, // 拆包会导致包数激增,不拆的话可能会出现单包过大的问题
            minChunks: 2, // 分包不能太细 不合并导致并发请求过多 浏览器有并发请求数量限制
            cacheGroups: {
                commons: {
                    test: /[\\/]node_modules[\\/]/,
                    name: 'vendors',
                    priority: 1,
                },
            },
        },
        // 将 optimization.runtimeChunk 设置为 true 或 'multiple'，会为每个入口添加一个只含有 runtime 的额外 chunk。
        runtimeChunk: {
            name: 'runtime',
        },
        minimize: true,
        minimizer: [
            new TerserPlugin({
                parallel: true, // 多线程
                extractComments: false, // 注释单独提取
                terserOptions: {
                    compress: {
                        drop_console: true, // 清除console输出
                    },
                    format: {
                        comments: false, // 清除注释
                    },
                    toplevel: true, //  声明提前
                    keep_classnames: true, // 类名不变
                },
            }),
            new CssMinimizerPlugin({
                minimizerOptions: {
                    preset: [
                        'default',
                        {
                            discardComments: { removeAll: true },
                        },
                    ],
                },
            }),
        ],
    },
};

const config = {
    mode: modeMap[NODE_ENV] || 'development',
    stats: 'errors-only',
    entry: './src/entry-client.js',
    infrastructureLogging: {
        level: 'error',
    },
    output: {
        path: path.resolve(__dirname, '../dist'),
        publicPath: '/',
        filename: isProd ? 'js/[name].[contenthash:6].bundle.js' : 'js/[name].bundle.js',
        chunkFilename: isProd ? 'js/chunk_[name]_[contenthash:6].js' : 'js/chunk_[name].js',
        // clean: true, // 在生成文件之前清空 output 目录
    },
    cache: {
        type: 'filesystem',
        buildDependencies: {
            config: [__filename],
        },
    },
    devServer: {
        hot: true,
        compress: true,
        host: '0.0.0.0',
        port: '9800',
        open: false,
        client: {
            logging: 'error',
            overlay: false,
        },
        liveReload: false,
        historyApiFallback: {
            rewrites: [
                {
                    from: /\//,
                    to: '/index.html',
                },
            ],
        },
    },
    resolve: {
        extensions: ['.js', '.vue', '.json', '.ts', '.html'],
        alias: {
            '@': path.resolve(__dirname, '../src'),
            vue$: 'vue/dist/vue.esm.js',
            axiosInstance: "@/utils/request-client.js"
        },
    },
    optimization: OptimizationMap[NODE_ENV],
    plugins: [
        new VueLoaderPlugin(),
        new WebpackBar({
			name: "Spa",
			color: "#9013FE"
		}),
        new HtmlWebpackPlugin({
            template: './public/index.spa.html',
            favicon: path.resolve(__dirname, '../public/favicon.ico'),
        }),
        new CopyPlugin({
            patterns: [{ from: path.resolve(__dirname, '../public'), to: 'public' }],
        }),
    ],
    module: {
        rules: [
            // vue-loader必须要在最外层,不能放入oneOf
            {
                test: /\.vue$/i,
                include: [path.resolve(__dirname, '../src')],
                exclude: /node_modules/,
                use: ['vue-loader'],
            },
            {
                oneOf: [
                    {
                        test: /\.(js|jsx)$/i,
                        exclude: /node_modules/,
                        use: ['thread-loader', 'babel-loader'],
                    },
                    {
                        test: /\.s[ac]ss$/i,
                        use: [
                            stylesHandler,
                            {
                                loader: 'css-loader',
                                options: {
                                    importLoaders: 2,
                                    modules: {
                                        mode: 'icss',
                                    },
                                },
                            },
                            'postcss-loader',
                            'sass-loader',
                        ],
                    },
                    {
                        test: /\.css$/i,
                        use: [
                            stylesHandler,
                            {
                                loader: 'css-loader',
                                options: {
                                    importLoaders: 1,
                                },
                            },
                            'postcss-loader',
                        ],
                    },
                    {
                        test: /\.(eot|svg|ttf|woff|woff2|png|jpg|gif)$/i,
                        type: 'asset',
                        parser: {
                            dataUrlCondition: {
                                maxSize: 10000,
                            },
                        },
                    },
                ],
            },
        ],
    },
};

module.exports = (env, argv) => {
    const APP_ENV = env.APP_ENV || 'development';
    config.plugins.push(
        new Dotenv({
            path: `./envVariable/client/.env.${APP_ENV}`,
        })
    );
    return new Promise((resolve) => {
        if (isProd) {
            config.plugins.push(
                new MiniCssExtractPlugin({
                    filename: 'styles/[name].[contenthash:6].css',
                }),
                new CompressionPlugin({
                    test: /\.(css|js)$/,
                    minRatio: 0.7,
                })
            );
            resolve(config);
        } else {
            config.devtool = 'cheap-module-source-map';
            portfinder.basePort = config.devServer.port;
            portfinder.getPort((err, port) => {
                if (err) {
                    reject(err);
                } else {
                    const LOCAL_IP = address.ip();
                    // publish the new Port, necessary for e2e tests
                    process.env.PORT = port;
                    // add port to devServer config,主要是这一步更新可用的端口
                    config.devServer.port = port;
                    // Add FriendlyErrorsPlugin
                    config.plugins.push(
                        new FriendlyErrorsWebpackPlugin({
                            compilationSuccessInfo: {
                                messages: [
                                    `  App running at:`,
                                    `  - Local:   ` + chalk.cyan(`http://localhost:${port}`),
                                    `  - Network: ` + chalk.cyan(`http://${LOCAL_IP}:${port}`),
                                ],
                            },
                            clearConsole: true,
                        })
                    );
                    resolve(config);
                }
            });
        }
    });
};

```

## 本地开发服务器

本地热更新与 `renderer` 刷新流程沿用 vue-hackernews-2.0 的做法。

已知，`SSR`的渲染关键在于`renderer.renderToString`，`renderer`是由`createBundleRenderer`创建而来，`createBundleRenderer`函数需要传入`bundle clientManifest`等实参。

因此，如果想在改完代码后拿到最新的内容，我们需要做以下事情：

1. 声明一个`renderer`
2. 构建server，在文件修改时重新编译，构建完成时想办法拿到最新的`bundle`
3. 构建client，在文件修改时重新编译，构建完成时想办法拿到最新的`clientManifest`
4. 拿到最新的`bundle`和`clientManifest`后，通过`createBundleRenderer`将`renderer`替换
5. 拿到最新的`renderer`执行`renderer.renderToString`

以上就是`setup-dev-server.js`要做的工作。

### setup-dev-server.js

通常情况下我们构建`webpack`，	通过命令指定一个`wepback`配置文件进行构建。

```shell
webpack --config webpack.xxx.config.js
```

除了上述方式，可以通过在`js`执行`webpack(AnyWebpackConfig)`来启动`webpack`构建。（via [Node 接口 | webpack 中文文档 (docschina.org)](https://webpack.docschina.org/api/node/#webpack)）

`setupDevServer`函数执行`webpack(clientConfig) webpack(serverConfig) `，在`compiler`钩子回调中，通过`outputFileSystem`（via [Node 接口 | webpack 中文文档 (docschina.org)](https://webpack.docschina.org/api/node/#custom-file-systems)）获取到最新的构建结果。

`renderer`的更新关键在`setupDevServer`传入的实参`cb`的执行时机。

> `webpack-dev-middleware`旧版本的`outputFileSystem`直接挂在`devMiddleware`下。
>
> 新版本通过`devMiddleware.context.outputFileSystem`访问

```js
const fs = require("fs")
const path = require("path")
const MFS = require("memory-fs")
const webpack = require("webpack")
const chokidar = require("chokidar")

module.exports = function setupDevServer(app, templatePath, cb, port) {
	const clientConfig = require("./webpack.client.config")({
		APP_ENV: "development"
	})
	const serverConfig = require("./webpack.server.config")(
		{
			APP_ENV: "development"
		},
		{ port }
	)

	const readFile = (fs, file) => {
		try {
			return fs.readFileSync(path.join(clientConfig.output.path, file), "utf-8")
		} catch (e) {
			console.log(e)
		}
	}

	let bundle
	let template
	let clientManifest

	let ready
	const readyPromise = new Promise((r) => {
		ready = r
	})
	const update = () => {
		if (bundle && clientManifest) {
			ready()
			cb(bundle, {
				template,
				clientManifest
			})
		}
	}

	// read template from disk and watch
	template = fs.readFileSync(templatePath, "utf-8")
	chokidar.watch(templatePath).on("change", () => {
		template = fs.readFileSync(templatePath, "utf-8")
		// console.log("index.ssr.html template updated.")
		update()
	})

	// modify client config to work with hot middleware
	clientConfig.entry.app = ["webpack-hot-middleware/client", clientConfig.entry.app]
	clientConfig.output.filename = "[name].js"
	clientConfig.plugins.push(new webpack.HotModuleReplacementPlugin(), new webpack.NoEmitOnErrorsPlugin())

	// dev middleware
	//  console.log("clientCompiler start")
	const clientCompiler = webpack(clientConfig)
	const devMiddleware = require("webpack-dev-middleware")(clientCompiler, {
		publicPath: clientConfig.output.publicPath,
		serverSideRender: true
		// noInfo: true
	})
	app.use(devMiddleware)
	clientCompiler.hooks.done.tap("done", (stats) => {
		stats = stats.toJson()
		stats.errors.forEach((err) => console.error(err))
		stats.warnings.forEach((err) => console.warn(err))
		if (stats.errors.length) return
		clientManifest = JSON.parse(readFile(devMiddleware.context.outputFileSystem, "vue-ssr-client-manifest.json"))
		update()
		// console.log("clientCompiler success")
	})

	// hot middleware
	app.use(require("webpack-hot-middleware")(clientCompiler, { heartbeat: 5000, log: false }))

	// watch and update server renderer
	// console.log("serverCompiler start")
	const serverCompiler = webpack(serverConfig)
	serverCompiler.hooks.done.tap("done", (stats) => {
		// console.log("serverCompiler success")
	})
	const mfs = new MFS()
	serverCompiler.outputFileSystem = mfs
	serverCompiler.watch({}, (err, stats) => {
		if (err) throw err
		stats = stats.toJson()
		if (stats.errors.length) return

		// read bundle generated by vue-ssr-webpack-plugin
		bundle = JSON.parse(readFile(mfs, "vue-ssr-server-bundle.json"))
		update()
	})

	return readyPromise
}

```

## 配置文件中的路径与 context

在配置文件中充斥着各种路径，有些是`./`，有些是`../`，在这里略作讨论讨论。

`webpackConfig.context`，这个配置项默认值是Node.js 进程的当前工作目录（就是`npm run`执行所在目录），如果在配置路径的过程中，没有使用到`path.resolve`之类的`api`，那么`./`指向的目录就是`webpackConfig.context`所指向的目录，而不是当前文件所在目录。

这个配置项直接影响其他配置项的最终结果。比如：

```js
entry: {
    home: './src/home.js',
    about: './src/about.js',
    contact: './src/contact.js',
}
```

还会影响到插件的配置路径，比如：

```js
// build/webpack.spa.config.js
new HtmlWebpackPlugin({
    // 注意两个路径的差异
    template: './public/index.spa.html',
    favicon: path.resolve(__dirname, '../public/favicon.ico'),
})
```

## 构建环境变量与业务环境变量

`NODE_ENV`在`webpack.config.js`中用于区分打包模式，可以用来配置`mode`配置项（只支持`'none' | 'development' | 'production'`），可以通过命令指定具体的值：

```shell
cross-env NODE_ENV=production
# 或
webpack --node-env production
# https://webpack.docschina.org/api/cli/#node-env
```

`NODE_ENV`并不直接影响业务代码中`process.env.NODE_ENV`。

业务代码中的环境变量依赖于`webpack.DefinePlugin`，你也可以通过`dotenv-webpack`加载指定的`env`文件注入环境变量。

```shell
webpack --node-env production --env APP_ENV=production
# https://webpack.docschina.org/api/cli/#env
```

```js
// wepback plugins
new Dotenv({
    path: `./envVariable/client/.env.${APP_ENV}`,
})
```

总的来说就是`NODE_ENV`决定了构建方式（`development|production`）涉及到代码压缩等内容，`APP_ENV`决定了业务代码用哪套环境变量（`dev|test|stag|uat|pre|prod`）。

## 数据预取阶段的 Cookie 与鉴权

场景：浏览器访问`http://localhost:9500/user`，若用户已登录则`/user`页面展示当前用户个人信息，如果未登录则重定向到`/login`

思考一个问题：在数据预取阶段（`Data Pre-Fetching `），`server`端向后端服务器发起的异步请求是否自动携带了浏览器传过来的`cookie`？

答案：没有自动携带。

为了实现在`server`端携带`client`端的`cookie`，我们需要：

1. `server`端收到浏览器的访问请求，将`req`对象中的`cookie`保存
2. 将`cookie`传入到`store`中
3. 在`store.dispatch`时将`cookie`传递给`axios`实例

下面给出简明代码（仅做描述思路）：

```js
// server.js
const express = require('express')
const app = express()
...
server.get('*', (req, res) => {
  const context = {
  	...,
    // 配置context.cookie
		cookie: req.headers.cookie
	}
	renderer.renderToString(context, (err, html) => {
    ...
	})
})

```

```js
// entry-server.js
import { createApp } from './app';
// 此处形参context就是renderer.renderToString传入的context实参
export default (context) => {
  return new Promise((resolve,reject) => {
    const { app, router, store } = createApp();
    // 保存cookie到store里
    if (context.cookie) {
      store.commit('cookieStore/save_cookie', context.cookie);
    }
  })
}

```

```js
// modules/cookieStore.js
export default {
	namespaced: true,
	state: () => ({
		cookie: ""
	}),
	mutations: {
		save_cookie(state, cookie) {
			state.cookie = cookie || ""
		}
	}
}
// store.js
import cookieStore from "./modules/cookie"
export function createStore() {
	return new Vuex.Store({
		state: {},
		getters: {
			cookie(state) {
				return state.cookieStore.cookie
			}
		},
		modules: {
			cookieStore
		}
	})
}

```

```vue
// vue
<template>
	<div>
		<div>userInfo</div>
		<div>name:{{ userInfo.name }}</div>
		<div>age:{{ userInfo.age }}</div>
		<div>gender:{{ userInfo.gender }}</div>
	</div>
</template>

<script>
import { createNamespacedHelpers } from "vuex"
const { mapState } = createNamespacedHelpers("userStore")

export default {
	name: "User",
	asyncData({ store }) {
		return store.dispatch("userStore/getUserInfo")
	},
	computed: {
		...mapState(["userInfo"])
	}
}
</script>

```

```js
// userStore.js
import { getUserInfo } from "@/api/user.js"

export default {
	namespaced: true,
	state: () => ({
		userInfo: {}
	}),
	getters: {},
	mutations: {
		setUserInfo(state, userInfo) {
			state.userInfo = userInfo
		}
	},
	actions: {
		getUserInfo({ rootGetters, commit }) {
			return getUserInfo({
				cookie: rootGetters.cookie
			})
				.then((res) => {
					commit("setUserInfo", res.data)
				})
		}
	}
}

```

预取阶段，`asyncData`执行`store.dispatch("userStore/getUserInfo")`，会在`actions`中向`api`函数传入`cookie`，通过这样处理，我们就能实现预取阶段发起的异步请求也携带了鉴权信息。

### Cookie 传参方式的取舍

在查阅和参考资料的过程中发现，很多方案是将`cookie`作为实参传入到`asyncData`中，即：

```js
// entry-server.js
Promise.all(
  matchedComponents.map((Component) => {
    if (Component.asyncData) {
      return Component.asyncData({
        store,
        route: router.currentRoute,
        // 注意此处cookie实参
        cookie: context.cookie
      });
    }
  })
)

```

```vue
// vue
<script>
import { createNamespacedHelpers } from "vuex"
const { mapState } = createNamespacedHelpers("userStore")

export default {
	name: "User",
	asyncData({ store,cookie }) {
		return store.dispatch("userStore/getUserInfo",cookie)
	},
	computed: {
		...mapState(["userInfo"])
	}
}
</script>

```

**这里讲讲我为什么没有将`cookie`作为`asyncData`的参数层层传递**

首先明确一点，在`client`端发起异步请求是不需要显式地设置`cookie`，因为它会自动携带。

只有`server`端**数据预取阶段**才需要手动设置`cookie`，而预取阶段发起的请求被声明在`actions`里，因此我们只需要在`actions`中的函数被调用时将`cookie`挂到`axios`实例上即可。

再一个原因就是，`asyncData`不仅在`server`端被调用，它也可以在`client`端的`beforeRouteUpdate`阶段被调用（参考示例工程的`/userlist`页面）。如果你显式的将`cookie`传给`asyncData`，那么你需要在**任何调用到`asyncData`的地方**，尽可能为它补充好参数。

也就是说你要在`client`端编写大量类似下面的代码：

```js
// 用心体会
asyncData({
  store:this.$store,
  route:this.$route,
  cookie:document.cookie
})
```

如果你不补充参数，或许会遭遇一个经典报错：`Uncaught TypeError: Cannot read properties of undefined`。

综上所述，因为`asyncData`调用位置的不确定性，我们应该尽可能避免给`asyncData`额外添加参数。

## 在通用代码中处理平台差异

[编写通用代码 | Vue SSR 指南 (vuejs.org)](https://v2.ssr.vuejs.org/zh/guide/universal.html#服务器上的数据响应)提到：

> 通用代码不可接受特定平台的 API，因此如果你的代码中，直接使用了像 `window` 或 `document`，这种仅浏览器可用的全局变量，则会在 Node.js 中执行时抛出错误，反之也是如此。

当你试图在通用代码中访问特定平台的API，必须要通过一些hack方式编写这段代码，才能使程序平稳地运行。

下面介绍三种方式：

### 使用 resolve.alias 拆分实现

```js
// app.js
import titleMixin from "titleMixin"
// webpack.server.config.js
resolve: {
  alias: {
    titleMixin:"@/utils/title-server.js"
  }
}
// webpack.client.config.js
resolve: {
  alias: {
    titleMixin:"@/utils/title-client.js"
  }
}
// title-client.js
function getTitle(vm) {
	const { title } = vm.$options
	if (title) {
		return typeof title === "function" ? title.call(vm) : title
	}
}

const clientTitleMixin = {
	mounted() {
		const title = getTitle(this)
		if (title) {
			document.title = ` ${title}`
		}
	}
}

export default clientTitleMixin
// title-server.js
function getTitle(vm) {
	const { title } = vm.$options
	if (title) {
		return typeof title === "function" ? title.call(vm) : title
	}
}

const serverTitleMixin = {
	created() {
		const title = getTitle(this)
		if (title) {
			this.$ssrContext.title = title
		}
	}
}

export default serverTitleMixin

```

在`SSR`工程中，`resolve.alias`也可以用来兼容只能在某个平台运行的第三方包（比如一些埋点api只能在客户端运行），你可以通过定义“假”的变量导出来使代码在另一个平台运行不报错。

### 使用 process.env 切换实现

```js
// app.js
import titleMixin from "@/utils/title"
Vue.mixin(titleMixin)
// utils/index.js
export const atClient = process.env.atClient == "true"
export const atServer = process.env.atServer == "true"
// utils/title.js
import { atServer } from "./index.js"
function getTitle(vm) {
	const { title } = vm.$options
	if (title) {
		return typeof title === "function" ? title.call(vm) : title
	}
}
const serverTitleMixin = {
	created() {
		const title = getTitle(this)
		if (title) {
			this.$ssrContext.title = title
		}
	}
}
const clientTitleMixin = {
	mounted() {
		const title = getTitle(this)
		if (title) {
			document.title = ` ${title}`
		}
	}
}
export default atServer ? serverTitleMixin : clientTitleMixin

```

### 使用 typeof 守卫区分运行环境

```js
function getTitle(vm) {
	const { title } = vm.$options
	if (title) {
		return typeof title === "function" ? title.call(vm) : title
	}
}

export const titleMixin = (function () {
	if (typeof window !== "undefined") {
		return {
			mounted() {
				const title = getTitle(this)
				if (title) {
					document.title = `Vue HN 2.0 | ${title}`
				}
			}
		}
	} else {
		return {
			created() {
				const title = getTitle(this)
				if (title) {
					this.$ssrContext.title = `Vue HN 2.0 | ${title}`
				}
			}
		}
	}
})()

```

不到万不得已不要在业务中直接使用`if`条件判断的方式进行平台区分。直接判断平台的条件代码越少，编写出来的代码可维护性越高。

## 通过缓存降低服务端压力

代码层面减少服务器负载的思路是：

1. 能在浏览器中缓存的部分，尽量在浏览器中缓存，这样能减少服务器流量（ServiceWorker）
2. 在一定时间内服务器渲染同样的页面内容应该有缓存复用机制避免重复渲染（页面级缓存），比如活动页
3. 纯静态组件或由`id`决定`DOM`内容的组件应该复用（组件级缓存）

### Service Worker

在规模不大的工程中运用`ServiceWorker`或许获得的收益远小于面对的风险，使用与否要根据业务场景、工程健壮性、测试充分度多种原因作出决定。

原因是`ServiceWorker`的缓存是在用户本地，当用户本地遇到因为`ServiceWorker`导致的问题时，开发人员可能都没办法复现，使用时要特别谨慎小心，最好留一个能快速无感关闭用户本地`ServiceWorker`的后门。

### 页面级缓存与组件级缓存

具体操作不再赘述，建议直接参考文档完成功能开发。[缓存 | Vue SSR 指南](https://v2.ssr.vuejs.org/zh/guide/caching.html#缓存)

**页面级缓存**

需要注意的是页面级缓存本质上是将网页的document文档内容当做字符串保存在内存中，这意味着页面会优先从缓存中读取内容（省去渲染过程，缓存服务器压力，加速响应），如果没有命中缓存才会流转到`renderToString`所处的中间件。因此，页面级缓存通常用于静态页面的缓存。

**组件级缓存**

组件级缓存通常也是用来缓存静态内容，比如`footer`、`article`等内容。

需要注意的是，当命中缓存时，组件的生命周期钩子将不会执行（有哪些钩子？请参考：[编写通用代码 | Vue SSR 指南](https://v2.ssr.vuejs.org/zh/guide/universal.html#组件生命周期钩子函数)）。如果组件启动缓存，应该避免在生命周期钩子中执行副作用代码。

当你使用`vue-meta`这个插件时，或许会碰到前面提到的副作用问题，可以参考以下链接：

[Vue meta and serverCacheKey (LRU Caching) · Issue #190 · nuxt/vue-meta](https://github.com/nuxt/vue-meta/issues/190)

## 总结

整体路线与 vue-hackernews-2.0 一致：双端打包、`setup-dev-server` 联动 `bundle` 与 `clientManifest`、Express 渲染 HTML。Webpack 5 侧主要是把 loader、优化与插件配置按环境拆开，并处理好 SSR 与 CSR 降级两条链路。后续若在团队内落地，建议把本文当作「配置地图」，与仓库中的实际文件对照阅读。

## 参考文献

1. [vue-hackernews-2.0](https://github.com/vuejs/vue-hackernews-2.0)
2. [What to do when Vue hydration fails，Lichter.io](https://blog.lichter.io/posts/vue-hydration-error/)
3. [Webpack 中文文档](https://webpack.docschina.org/)
4. [Vue 2 服务器端渲染指南](https://v2.ssr.vuejs.org/zh/)
5. [Vue SSR 高阶指南，掘金](https://juejin.cn/post/6844903669922529287)
6. [SSR 项目的 Service Worker 最佳实践，哔哩哔哩](https://www.bilibili.com/video/BV1mt411q79H)
7. [在 Vue SSR 中使用 Service Worker，知乎专栏](https://zhuanlan.zhihu.com/p/31630322)
8. [SSR 架构项目实现离线可用（思路与案例），知乎专栏](https://zhuanlan.zhihu.com/p/30791448)
9. [pwa-lesson-demo：lesson-4-ssr-service-worker，GitHub](https://github.com/lavas-project/pwa-lesson-demo/tree/master/phase-2/lesson-4-ssr-service-worker)
10. [谨慎处理 Service Worker 的更新，掘金](https://juejin.cn/post/6844903792522035208)