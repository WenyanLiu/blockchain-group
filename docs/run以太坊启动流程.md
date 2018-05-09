    下面我们来简单的介绍这几个命令的使用场景
    
    abigen-- 一个源代码生成器，它将Ethereum智能合约定义(代码) 转换为易于使用的、编译时类型安全的Go package。 如果合约字节码也available的话，它可以在普通的Ethereum智能合约ABI上扩展功能。 同时也能编译Solidity源文件，使开发更加精简。
    bootnod–此Ethereum客户端实现的剥离版本只参与 网络节点发现 协议，但不运行任何更高级别的应用协议。 它可以用作轻量级引导节点，以帮助在私有网络中查找peers。
    evm-- 能够在可配置环境和执行模式下运行字节码片段的Developer utility版本的的EVM（Ethereum Virtual Machine）。 其目的是允许对EVM操作码进行封装，细粒度的调试。
    faucet–暂时不知道其使用场景，其help没有相关的解释，后续看下源码再来补充。
    geth–主要Ethereum CLI客户端。它是Ethereum网络（以太坊主网，测试网络或私有网）的入口点，使用此命令可以使节点作为full node（默认），或者archive node（保留所有历史状态）或light node（检索数据实时）运行。 其他进程可以通过暴露在HTTP，WebSocket和/或IPC传输之上的JSON RPC端点作为通向Ethereum网络的网关使用。
    puppeth–暂时不知道其含义，后续补充。
    rlpdump–开发者通用工具，用来把二进制RLP (Recursive Length Prefix) (Ethereum 协议中用于网络及一致性的数据编码) 转换成用户友好的分层表示。
    swarm–swarm守护进程和工具，这是swarm网络的进入点。

接下来我们从以太坊最重要的geth命令入手，来分析以太坊的启动流程。
首先，在命令行输入一下命令打开控制台：

build/bin/geth --datadir=./dev/data0 --networkid 1 console

接下来我们来看启动入口main函数，它位于/cmd/geth/main.go文件中，main函数的初始化函数代码如下：
```
func init() {
	// Initialize the CLI app and start Geth
	app.Action = geth
	app.HideVersion = true // we have a command to print the version
	app.Copyright = "Copyright 2013-2017 The go-ethereum Authors"
	app.Commands = []cli.Command{
		// See chaincmd.go:
		initCommand,
		importCommand,
		exportCommand,
		removedbCommand,
		dumpCommand,
		// See monitorcmd.go:
		monitorCommand,
		// See accountcmd.go:
		accountCommand,
		walletCommand,
		// See consolecmd.go:
		consoleCommand,
		attachCommand,
		javascriptCommand,
		// See misccmd.go:
		makedagCommand,
		versionCommand,
		bugCommand,
		licenseCommand,
		// See config.go
		dumpConfigCommand,
	}
	app.Flags = append(app.Flags, nodeFlags...)
	app.Flags = append(app.Flags, rpcFlags...)
	app.Flags = append(app.Flags, consoleFlags...)
	app.Flags = append(app.Flags, debug.Flags...)
	app.Flags = append(app.Flags, whisperFlags...)
	app.Before = func(ctx *cli.Context) error {
		runtime.GOMAXPROCS(runtime.NumCPU())
		if err := debug.Setup(ctx); err != nil {
			return err
		}
		// Start system runtime metrics collection
		go metrics.CollectProcessMetrics(3 * time.Second)
		utils.SetupNetwork(ctx)
		return nil
	}
	app.After = func(ctx *cli.Context) error {
		debug.Exit()
		console.Stdin.Close() // Resets terminal mode.
		return nil
	}
}
```
init函数主要是做了一些初始化的工作，其中比较重要的有三个地方，app.Action=geth, app.Commands中consoleCommand， 以及App.Before指向的匿名函数，后续使用到的时候我们再来分析。我们再来看看main函数：

```
func main() {
	if err := app.Run(os.Args); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```
main函数的实现很简单，仅仅调用了app.Run函数，如果调用异常，则退出。接下里我们看看app.Run函数的逻辑处理

```
func (a *App) Run(arguments []string) (err error) {
	a.Setup()

	// handle the completion flag separately from the flagset since
	// completion could be attempted after a flag, but before its value was put
	// on the command line. this causes the flagset to interpret the completion
	// flag name as the value of the flag before it which is undesirable
	// note that we can only do this because the shell autocomplete function
	// always appends the completion flag at the end of the command
	shellComplete, arguments := checkShellCompleteFlag(a, arguments)

	// parse flags
	set, err := flagSet(a.Name, a.Flags)
	if err != nil {
		return err
	}

	set.SetOutput(ioutil.Discard)
	err = set.Parse(arguments[1:])
	nerr := normalizeFlags(a.Flags, set)
	context := NewContext(a, set, nil)
	if nerr != nil {
		fmt.Fprintln(a.Writer, nerr)
		ShowAppHelp(context)
		return nerr
	}
	context.shellComplete = shellComplete

	if checkCompletions(context) {
		return nil
	}

	if err != nil {
		if a.OnUsageError != nil {
			err := a.OnUsageError(context, err, false)
			HandleExitCoder(err)
			return err
		}
		fmt.Fprintf(a.Writer, "%s %s\n\n", "Incorrect Usage.", err.Error())
		ShowAppHelp(context)
		return err
	}

	if !a.HideHelp && checkHelp(context) {
		ShowAppHelp(context)
		return nil
	}

	if !a.HideVersion && checkVersion(context) {
		ShowVersion(context)
		return nil
	}

	if a.After != nil {
		defer func() {
			if afterErr := a.After(context); afterErr != nil {
				if err != nil {
					err = NewMultiError(err, afterErr)
				} else {
					err = afterErr
				}
			}
		}()
	}

	if a.Before != nil {
		beforeErr := a.Before(context)
		if beforeErr != nil {
			fmt.Fprintf(a.Writer, "%v\n\n", beforeErr)
			ShowAppHelp(context)
			HandleExitCoder(beforeErr)
			err = beforeErr
			return err
		}
	}

	args := context.Args()
	if args.Present() {
		name := args.First()
		c := a.Command(name)
		if c != nil {
			return c.Run(context)
		}
	}

	if a.Action == nil {
		a.Action = helpCommand.Action
	}

	// Run default Action
	err = HandleAction(a.Action, context)

	HandleExitCoder(err)
	return err
}

```
a.Setup仅仅是做了些简单的处理，比如相关的Auther、Email、重新创建Command切片等等.接下来我们看看下面的这个if判断
```
if args.Present() {
		name := args.First()
		c := a.Command(name)
		if c != nil {
			return c.Run(context)
		}
	}
```
由于我们前面在控制台输入的命令（build/bin/geth --datadir=./dev/data0 --networkid 1 console）长度不为0，因此执行命令操作，此时的命令其实就是我们的console命令。
```
c.Run(context)
```
接下来我们看看Run方法，Run方法的代码如下
```
func (c Command) Run(ctx *Context) (err error) {
	if len(c.Subcommands) > 0 {
		return c.startApp(ctx)
	}

	if !c.HideHelp && (HelpFlag != BoolFlag{}) {
		// append help to flags
		c.Flags = append(
			c.Flags,
			HelpFlag,
		)
	}

	set, err := flagSet(c.Name, c.Flags)
	if err != nil {
		return err
	}
	set.SetOutput(ioutil.Discard)

	if c.SkipFlagParsing {
		err = set.Parse(append([]string{"--"}, ctx.Args().Tail()...))
	} else if !c.SkipArgReorder {
		firstFlagIndex := -1
		terminatorIndex := -1
		for index, arg := range ctx.Args() {
			if arg == "--" {
				terminatorIndex = index
				break
			} else if arg == "-" {
				// Do nothing. A dash alone is not really a flag.
				continue
			} else if strings.HasPrefix(arg, "-") && firstFlagIndex == -1 {
				firstFlagIndex = index
			}
		}

		if firstFlagIndex > -1 {
			args := ctx.Args()
			regularArgs := make([]string, len(args[1:firstFlagIndex]))
			copy(regularArgs, args[1:firstFlagIndex])

			var flagArgs []string
			if terminatorIndex > -1 {
				flagArgs = args[firstFlagIndex:terminatorIndex]
				regularArgs = append(regularArgs, args[terminatorIndex:]...)
			} else {
				flagArgs = args[firstFlagIndex:]
			}

			err = set.Parse(append(flagArgs, regularArgs...))
		} else {
			err = set.Parse(ctx.Args().Tail())
		}
	} else {
		err = set.Parse(ctx.Args().Tail())
	}

	nerr := normalizeFlags(c.Flags, set)
	if nerr != nil {
		fmt.Fprintln(ctx.App.Writer, nerr)
		fmt.Fprintln(ctx.App.Writer)
		ShowCommandHelp(ctx, c.Name)
		return nerr
	}

	context := NewContext(ctx.App, set, ctx)
	if checkCommandCompletions(context, c.Name) {
		return nil
	}

	if err != nil {
		if c.OnUsageError != nil {
			err := c.OnUsageError(ctx, err, false)
			HandleExitCoder(err)
			return err
		}
		fmt.Fprintln(ctx.App.Writer, "Incorrect Usage:", err.Error())
		fmt.Fprintln(ctx.App.Writer)
		ShowCommandHelp(ctx, c.Name)
		return err
	}

	if checkCommandHelp(context, c.Name) {
		return nil
	}

	if c.After != nil {
		defer func() {
			afterErr := c.After(context)
			if afterErr != nil {
				HandleExitCoder(err)
				if err != nil {
					err = NewMultiError(err, afterErr)
				} else {
					err = afterErr
				}
			}
		}()
	}

	if c.Before != nil {
		err = c.Before(context)
		if err != nil {
			fmt.Fprintln(ctx.App.Writer, err)
			fmt.Fprintln(ctx.App.Writer)
			ShowCommandHelp(ctx, c.Name)
			HandleExitCoder(err)
			return err
		}
	}

	if c.Action == nil {
		c.Action = helpSubcommand.Action
	}

	context.Command = c
	err = HandleAction(c.Action, context)

	if err != nil {
		HandleExitCoder(err)
	}
	return err
}
```
该主要是设置flag、解析输入的命令行参数、创建全局的context、将当前命令保存到全局context中,接下来调用HandleAction来处理命令，HandleAction的函数实现如下：
```
func HandleAction(action interface{}, context *Context) (err error) {
	if a, ok := action.(ActionFunc); ok {
		return a(context)
	} else if a, ok := action.(func(*Context) error); ok {
		return a(context)
	} else if a, ok := action.(func(*Context)); ok { // deprecated function signature
		a(context)
		return nil
	} else {
		return errInvalidActionType
	}
}
```
action的类型是「func(*Context) error」，此时将执行a(context)方法，那么此时调用那个Action呢，答案就是我们前面提到的App.init()初始化命令时的	app.Action = geth，接下来我们来看看cmd/geth/main.go中的geth：
```
func geth(ctx *cli.Context) error {
	node := makeFullNode(ctx)
	startNode(ctx, node)
	node.Wait()
	return nil
}
```

该函数非常重要，主要完成以下几件事情

    首先会创建一个节点、makeFullNode(ctx)
    同时启动该节点startNode(ctx,node)
    node.Wait()监听stop通道
    

首先我们来看该Node是如何创建的。makeFullNode函数的实现如下：
```
func makeFullNode(ctx *cli.Context) *node.Node {
	stack, cfg := makeConfigNode(ctx)

	utils.RegisterEthService(stack, &cfg.Eth)

	// Whisper must be explicitly enabled by specifying at least 1 whisper flag or in dev mode
	shhEnabled := enableWhisper(ctx)
	shhAutoEnabled := !ctx.GlobalIsSet(utils.WhisperEnabledFlag.Name) && ctx.GlobalIsSet(utils.DevModeFlag.Name)
	if shhEnabled || shhAutoEnabled {
		if ctx.GlobalIsSet(utils.WhisperMaxMessageSizeFlag.Name) {
			cfg.Shh.MaxMessageSize = uint32(ctx.Int(utils.WhisperMaxMessageSizeFlag.Name))
		}
		if ctx.GlobalIsSet(utils.WhisperMinPOWFlag.Name) {
			cfg.Shh.MinimumAcceptedPOW = ctx.Float64(utils.WhisperMinPOWFlag.Name)
		}
		utils.RegisterShhService(stack, &cfg.Shh)
	}

	// Add the Ethereum Stats daemon if requested.
	if cfg.Ethstats.URL != "" {
		utils.RegisterEthStatsService(stack, cfg.Ethstats.URL)
	}

	// Add the release oracle service so it boots along with node.
	if err := stack.Register(func(ctx *node.ServiceContext) (node.Service, error) {
		config := release.Config{
			Oracle: relOracle,
			Major:  uint32(params.VersionMajor),
			Minor:  uint32(params.VersionMinor),
			Patch:  uint32(params.VersionPatch),
		}
		commit, _ := hex.DecodeString(gitCommit)
		copy(config.Commit[:], commit)
		return release.NewReleaseService(ctx, config)
	}); err != nil {
		utils.Fatalf("Failed to register the Geth release oracle service: %v", err)
	}
	return stack
}
```

该函数首先创建一个Node，然后注册一个Ethereum Service，我们继续分析Node是如何创建的，makeConfigNode的函数实现逻辑如下：
```
func makeConfigNode(ctx *cli.Context) (*node.Node, gethConfig) {
	// Load defaults.
	cfg := gethConfig{
		Eth:  eth.DefaultConfig,
		Shh:  whisper.DefaultConfig,
		Node: defaultNodeConfig(),
	}

	// Load config file.
	if file := ctx.GlobalString(configFileFlag.Name); file != "" {
		if err := loadConfig(file, &cfg); err != nil {
			utils.Fatalf("%v", err)
		}
	}

	// Apply flags.
	utils.SetNodeConfig(ctx, &cfg.Node)
	stack, err := node.New(&cfg.Node)
	if err != nil {
		utils.Fatalf("Failed to create the protocol stack: %v", err)
	}
	utils.SetEthConfig(ctx, stack, &cfg.Eth)
	if ctx.GlobalIsSet(utils.EthStatsURLFlag.Name) {
		cfg.Ethstats.URL = ctx.GlobalString(utils.EthStatsURLFlag.Name)
	}

	utils.SetShhConfig(ctx, stack, &cfg.Shh)

	return stack, cfg
}
```
首先初始化gethConfig的变量，接下来调用node.New方法来创建一个Node，我们来看看New函数的实现：
```
func New(conf *Config) (*Node, error) {
	// Copy config and resolve the datadir so future changes to the current
	// working directory don't affect the node.
	confCopy := *conf
	conf = &confCopy
	if conf.DataDir != "" {
		absdatadir, err := filepath.Abs(conf.DataDir)
		if err != nil {
			return nil, err
		}
		conf.DataDir = absdatadir
	}
	// Ensure that the instance name doesn't cause weird conflicts with
	// other files in the data directory.
	if strings.ContainsAny(conf.Name, `/\`) {
		return nil, errors.New(`Config.Name must not contain '/' or '\'`)
	}
	if conf.Name == datadirDefaultKeyStore {
		return nil, errors.New(`Config.Name cannot be "` + datadirDefaultKeyStore + `"`)
	}
	if strings.HasSuffix(conf.Name, ".ipc") {
		return nil, errors.New(`Config.Name cannot end in ".ipc"`)
	}
	// Ensure that the AccountManager method works before the node has started.
	// We rely on this in cmd/geth.
	am, ephemeralKeystore, err := makeAccountManager(conf)
	if err != nil {
		return nil, err
	}
	// Note: any interaction with Config that would create/touch files
	// in the data directory or instance directory is delayed until Start.
	return &Node{
		accman:            am,
		ephemeralKeystore: ephemeralKeystore,
		config:            conf,
		serviceFuncs:      []ServiceConstructor{},
		ipcEndpoint:       conf.IPCEndpoint(),
		httpEndpoint:      conf.HTTPEndpoint(),
		wsEndpoint:        conf.WSEndpoint(),
		eventmux:          new(event.TypeMux),
	}, nil
}
```

New函数首先将保存数据的目录转换成绝对路径，然后对配置进行相关的验证，确认是否合法，接下来创建一个AccountManager实例，然后创建一个Node节点，总体来说其内部的逻辑比较的简单。由于AccountManager负责Account的相关管理操作且后续经常使用，接下来我们看看是AccountManager的具体逻辑实现。AccountManager位于/node/config.go文件中，其实现如下：
```
func makeAccountManager(conf *Config) (*accounts.Manager, string, error) {
	scryptN := keystore.StandardScryptN
	scryptP := keystore.StandardScryptP
	if conf.UseLightweightKDF {
		scryptN = keystore.LightScryptN
		scryptP = keystore.LightScryptP
	}

	var (
		keydir    string
		ephemeral string
		err       error
	)
	switch {
	case filepath.IsAbs(conf.KeyStoreDir):
		keydir = conf.KeyStoreDir
	case conf.DataDir != "":
		if conf.KeyStoreDir == "" {
			keydir = filepath.Join(conf.DataDir, datadirDefaultKeyStore)
		} else {
			keydir, err = filepath.Abs(conf.KeyStoreDir)
		}
	case conf.KeyStoreDir != "":
		keydir, err = filepath.Abs(conf.KeyStoreDir)
	default:
		// There is no datadir.
		keydir, err = ioutil.TempDir("", "go-ethereum-keystore")
		ephemeral = keydir
	}
	if err != nil {
		return nil, "", err
	}
	if err := os.MkdirAll(keydir, 0700); err != nil {
		return nil, "", err
	}
	// Assemble the account manager and supported backends
	backends := []accounts.Backend{
		keystore.NewKeyStore(keydir, scryptN, scryptP),
	}
	if !conf.NoUSB {
		if ledgerhub, err := usbwallet.NewLedgerHub(); err != nil {
			log.Warn(fmt.Sprintf("Failed to start Ledger hub, disabling: %v", err))
		} else {
			backends = append(backends, ledgerhub)
		}
	}
	return accounts.NewManager(backends...), ephemeral, nil
}
```

由于在上面，我们已经将保存数据的目录转换成了绝对路径（由于在命令行，我们已经指定了datadir且存在，如果不存在将创建一个默认的go-ethereum-keystore来保存相关信息），接下来我们来看keystore.NewKeyStore函数。该函数位于/accounts/keystore/keystore.go文件中，其实现如下：

```
func NewKeyStore(keydir string, scryptN, scryptP int) *KeyStore {
	keydir, _ = filepath.Abs(keydir)
	ks := &KeyStore{storage: &keyStorePassphrase{keydir, scryptN, scryptP}}
	ks.init(keydir)
	return ks
}
```
首先转换成绝对路径（前面已经转换，应该没有必要再次检测），接下来我们看看
```
ks.init(keydir)
```
方法的实现，其实现如下：
```
func (ks *KeyStore) init(keydir string) {
	// Lock the mutex since the account cache might call back with events
	ks.mu.Lock()
	defer ks.mu.Unlock()

	// Initialize the set of unlocked keys and the account cache
	ks.unlocked = make(map[common.Address]*unlocked)
	ks.cache, ks.changes = newAccountCache(keydir)

	// TODO: In order for this finalizer to work, there must be no references
	// to ks. addressCache doesn't keep a reference but unlocked keys do,
	// so the finalizer will not trigger until all timed unlocks have expired.
	runtime.SetFinalizer(ks, func(m *KeyStore) {
		m.cache.close()
	})
	// Create the initial list of wallets from the cache
	accs := ks.cache.accounts()
	ks.wallets = make([]accounts.Wallet, len(accs))
	for i := 0; i < len(accs); i++ {
		ks.wallets[i] = &keystoreWallet{account: accs[i], keystore: ks}
	}
}
```
该方法首先从缓存中（如果缓存存在则直接获取，否则从datadir中读取）获取Account，然后保存在keystore的钱包中，我们主要来看看ks.cache.accounts()函数，其实现逻辑如下：
```
func (ac *accountCache) accounts() []accounts.Account {
	ac.maybeReload()
	ac.mu.Lock()
	defer ac.mu.Unlock()
	cpy := make([]accounts.Account, len(ac.all))
	copy(cpy, ac.all)
	return cpy
}
```
如果账户数据未被加载到内存中，则首先加载进来，然后copy一份，防止外面对缓存的账户做修改，我们来看看ac.maybeReload是如何加载账户信息的。其实现如下：

```
func (ac *accountCache) maybeReload() {
	ac.mu.Lock()
	defer ac.mu.Unlock()

	if ac.watcher.running {
		return // A watcher is running and will keep the cache up-to-date.
	}
	if ac.throttle == nil {
		ac.throttle = time.NewTimer(0)
	} else {
		select {
		case <-ac.throttle.C:
		default:
			return // The cache was reloaded recently.
		}
	}
	ac.watcher.start()
	ac.reload()
	ac.throttle.Reset(minReloadInterval)
}
```
首先使用goroutine启动一个watcher来监测keystore目录，防止其变化。接下来调用reload方法加载账户信息，reload方法的实现如下：
```
func (ac *accountCache) reload() {
	accounts, err := ac.scan()
	if err != nil {
		log.Debug("Failed to reload keystore contents", "err", err)
	}
	ac.all = accounts
	sort.Sort(ac.all)
	for k := range ac.byAddr {
		delete(ac.byAddr, k)
	}
	for _, a := range accounts {
		ac.byAddr[a.Address] = append(ac.byAddr[a.Address], a)
	}
	select {
	case ac.notify <- struct{}{}:
	default:
	}
	log.Debug("Reloaded keystore contents", "accounts", len(ac.all))
}
```
reload方法首先加载Account信息，然后通过channel的方式通知加载结束。Scan方法就是最终从文件中加载账户信息的地方，其实现如下：
```
func (ac *accountCache) scan() ([]accounts.Account, error) {
	files, err := ioutil.ReadDir(ac.keydir)
	if err != nil {
		return nil, err
	}

	var (
		buf     = new(bufio.Reader)
		addrs   []accounts.Account
		keyJSON struct {
			Address string `json:"address"`
		}
	)
	for _, fi := range files {
		path := filepath.Join(ac.keydir, fi.Name())
		if skipKeyFile(fi) {
			log.Trace("Ignoring file on account scan", "path", path)
			continue
		}
		logger := log.New("path", path)

		fd, err := os.Open(path)
		if err != nil {
			logger.Trace("Failed to open keystore file", "err", err)
			continue
		}
		buf.Reset(fd)
		// Parse the address.
		keyJSON.Address = ""
		err = json.NewDecoder(buf).Decode(&keyJSON)
		addr := common.HexToAddress(keyJSON.Address)
		switch {
		case err != nil:
			logger.Debug("Failed to decode keystore key", "err", err)
		case (addr == common.Address{}):
			logger.Debug("Failed to decode keystore key", "err", "missing or zero address")
		default:
			addrs = append(addrs, accounts.Account{Address: addr, URL: accounts.URL{Scheme: KeyStoreScheme, Path: path}})
		}
		fd.Close()
	}
	return addrs, err
}
```
该方法首先获取该目录下的文件，然后读取文件内容，JSON解析其address字段。下面是我的datadir/keystore中文件的内容,格式化后如下：
```
{
	"address":"a8f8687d0da839cef651cc0d2dc41bb2dc796293",
	"crypto":{
		"cipher":"aes-128-ctr",
		"ciphertext":"142da6b140f5f4ddd48205997cfc6257312dde63e227579bdc		a427a24661b9f8",
		"cipherparams":{
		"iv":"8d45e1d8b9fbd72fa40799c3a2ca1d22"
	},
	"kdf":"scrypt",
	"kdfparams":{
		"dklen":32,
		"n":262144,
		"p":1,
		"r":8,
		"salt":"68d9776e818a5c1a92982752df1b3c66fc54a804d508a926519d33e046158959"
		},
	"mac":"898b1f94fda7731215e521ea7ca7eb7d9afd9b8255c6dccd244c78af88123e7e"
	},
	"id":"dfefe914-e328-4f7f-89a7-a85689b8e855",
	"version":3
}
```
从json可以看出我这里只有一个账户的信息，因为以太坊对于每个用户都创建一个单独的文件来保存其信息。 对于账户信息的加载，AccountManager的创建我们分析完了，同时整个Node也创建完成，此时我们来看以太坊是如何启动一个节点的。此时回到main函数，startNode函数的代码如下：
```
func startNode(ctx *cli.Context, stack *node.Node) {
	// Start up the node itself
	utils.StartNode(stack)

	// Unlock any account specifically requested
	ks := stack.AccountManager().Backends(keystore.KeyStoreType)[0].(*keystore.KeyStore)

	passwords := utils.MakePasswordList(ctx)
	unlocks := strings.Split(ctx.GlobalString(utils.UnlockedAccountFlag.Name), ",")
	for i, account := range unlocks {
		if trimmed := strings.TrimSpace(account); trimmed != "" {
			unlockAccount(ctx, ks, trimmed, i, passwords)
		}
	}
	// Register wallet event handlers to open and auto-derive wallets
	events := make(chan accounts.WalletEvent, 16)
	stack.AccountManager().Subscribe(events)

	go func() {
		// Create an chain state reader for self-derivation
		rpcClient, err := stack.Attach()
		if err != nil {
			utils.Fatalf("Failed to attach to self: %v", err)
		}
		stateReader := ethclient.NewClient(rpcClient)

		// Open and self derive any wallets already attached
		for _, wallet := range stack.AccountManager().Wallets() {
			if err := wallet.Open(""); err != nil {
				log.Warn("Failed to open wallet", "url", wallet.URL(), "err", err)
			} else {
				wallet.SelfDerive(accounts.DefaultBaseDerivationPath, stateReader)
			}
		}
		// Listen for wallet event till termination
		for event := range events {
			if event.Arrive {
				if err := event.Wallet.Open(""); err != nil {
					log.Warn("New wallet appeared, failed to open", "url", event.Wallet.URL(), "err", err)
				} else {
					log.Info("New wallet appeared", "url", event.Wallet.URL(), "status", event.Wallet.Status())
					event.Wallet.SelfDerive(accounts.DefaultBaseDerivationPath, stateReader)
				}
			} else {
				log.Info("Old wallet dropped", "url", event.Wallet.URL())
				event.Wallet.Close()
			}
		}
	}()
	// Start auxiliary services if enabled
	if ctx.GlobalBool(utils.MiningEnabledFlag.Name) {
		// Mining only makes sense if a full Ethereum node is running
		var ethereum *eth.Ethereum
		if err := stack.Service(&ethereum); err != nil {
			utils.Fatalf("ethereum service not running: %v", err)
		}
		// Use a reduced number of threads if requested
		if threads := ctx.GlobalInt(utils.MinerThreadsFlag.Name); threads > 0 {
			type threaded interface {
				SetThreads(threads int)
			}
			if th, ok := ethereum.Engine().(threaded); ok {
				th.SetThreads(threads)
			}
		}
		// Set the gas price to the limits from the CLI and start mining
		ethereum.TxPool().SetGasPrice(utils.GlobalBig(ctx, utils.GasPriceFlag.Name))
		if err := ethereum.StartMining(true); err != nil {
			utils.Fatalf("Failed to start mining: %v", err)
		}
	}
}

```
该函数内部首先将节点启动起来，然后建立跟RPC Server建立一个连接，用于RPC通信。我们来看看节点如何启动起来的。/cmd/utils/cmd.go文件如下：
```
func StartNode(stack *node.Node) {
	if err := stack.Start(); err != nil {
		Fatalf("Error starting protocol stack: %v", err)
	}
	go func() {
		sigc := make(chan os.Signal, 1)
		signal.Notify(sigc, os.Interrupt)
		defer signal.Stop(sigc)
		<-sigc
		log.Info("Got interrupt, shutting down...")
		go stack.Stop()
		for i := 10; i > 0; i-- {
			<-sigc
			if i > 1 {
				log.Warn("Already shutting down, interrupt more to panic.", "times", i-1)
			}
		}
		debug.Exit() // ensure trace and CPU profile data is flushed.
		debug.LoudPanic("boom")
	}()
}
```
该函数调用了Node.go中的start方法，其实现如下：
```
func (n *Node) Start() error {
	n.lock.Lock()
	defer n.lock.Unlock()

	// Short circuit if the node's already running
	if n.server != nil {
		return ErrNodeRunning
	}
	if err := n.openDataDir(); err != nil {
		return err
	}

	// Initialize the p2p server. This creates the node key and
	// discovery databases.
	n.serverConfig = n.config.P2P
	n.serverConfig.PrivateKey = n.config.NodeKey()
	n.serverConfig.Name = n.config.NodeName()
	if n.serverConfig.StaticNodes == nil {
		n.serverConfig.StaticNodes = n.config.StaticNodes()
	}
	if n.serverConfig.TrustedNodes == nil {
		n.serverConfig.TrustedNodes = n.config.TrustedNodes()
	}
	if n.serverConfig.NodeDatabase == "" {
		n.serverConfig.NodeDatabase = n.config.NodeDB()
	}
	running := &p2p.Server{Config: n.serverConfig}
	log.Info("Starting peer-to-peer node", "instance", n.serverConfig.Name)

	// Otherwise copy and specialize the P2P configuration
	services := make(map[reflect.Type]Service)
	for _, constructor := range n.serviceFuncs {
		// Create a new context for the particular service
		ctx := &ServiceContext{
			config:         n.config,
			services:       make(map[reflect.Type]Service),
			EventMux:       n.eventmux,
			AccountManager: n.accman,
		}
		for kind, s := range services { // copy needed for threaded access
			ctx.services[kind] = s
		}
		// Construct and save the service
		service, err := constructor(ctx)
		if err != nil {
			return err
		}
		kind := reflect.TypeOf(service)
		if _, exists := services[kind]; exists {
			return &DuplicateServiceError{Kind: kind}
		}
		services[kind] = service
	}
	// Gather the protocols and start the freshly assembled P2P server
	for _, service := range services {
		running.Protocols = append(running.Protocols, service.Protocols()...)
	}
	if err := running.Start(); err != nil {
		if errno, ok := err.(syscall.Errno); ok && datadirInUseErrnos[uint(errno)] {
			return ErrDatadirUsed
		}
		return err
	}
	// Start each of the services
	started := []reflect.Type{}
	for kind, service := range services {
		// Start the next service, stopping all previous upon failure
		if err := service.Start(running); err != nil {
			for _, kind := range started {
				services[kind].Stop()
			}
			running.Stop()

			return err
		}
		// Mark the service started for potential cleanup
		started = append(started, kind)
	}
	// Lastly start the configured RPC interfaces
	if err := n.startRPC(services); err != nil {
		for _, service := range services {
			service.Stop()
		}
		running.Stop()
		return err
	}
	// Finish initializing the startup
	n.services = services
	n.server = running
	n.stop = make(chan struct{})

	return nil
}
```
该方法首先打开datadir目录，接着初始化serverConfig的相关配置，接着创建一个p2p.server的一个变量，然后启动该节点，在接着把RPC的Service启动起来，我们先看看节点的启动。/p2p/server.go中Start方法如下：
```
func (srv *Server) Start() (err error) {
	srv.lock.Lock()
	defer srv.lock.Unlock()
	if srv.running {
		return errors.New("server already running")
	}
	srv.running = true
	log.Info("Starting P2P networking")

	// static fields
	if srv.PrivateKey == nil {
		return fmt.Errorf("Server.PrivateKey must be set to a non-nil key")
	}
	if srv.newTransport == nil {
		srv.newTransport = newRLPX
	}
	if srv.Dialer == nil {
		srv.Dialer = &net.Dialer{Timeout: defaultDialTimeout}
	}
	srv.quit = make(chan struct{})
	srv.addpeer = make(chan *conn)
	srv.delpeer = make(chan peerDrop)
	srv.posthandshake = make(chan *conn)
	srv.addstatic = make(chan *discover.Node)
	srv.removestatic = make(chan *discover.Node)
	srv.peerOp = make(chan peerOpFunc)
	srv.peerOpDone = make(chan struct{})

	// node table
	if !srv.NoDiscovery {
		ntab, err := discover.ListenUDP(srv.PrivateKey, srv.ListenAddr, srv.NAT, srv.NodeDatabase, srv.NetRestrict)
		if err != nil {
			return err
		}
		if err := ntab.SetFallbackNodes(srv.BootstrapNodes); err != nil {
			return err
		}
		srv.ntab = ntab
	}

	if srv.DiscoveryV5 {
		ntab, err := discv5.ListenUDP(srv.PrivateKey, srv.DiscoveryV5Addr, srv.NAT, "", srv.NetRestrict) //srv.NodeDatabase)
		if err != nil {
			return err
		}
		if err := ntab.SetFallbackNodes(srv.BootstrapNodesV5); err != nil {
			return err
		}
		srv.DiscV5 = ntab
	}

	dynPeers := (srv.MaxPeers + 1) / 2
	if srv.NoDiscovery {
		dynPeers = 0
	}
	dialer := newDialState(srv.StaticNodes, srv.BootstrapNodes, srv.ntab, dynPeers, srv.NetRestrict)

	// handshake
	srv.ourHandshake = &protoHandshake{Version: baseProtocolVersion, Name: srv.Name, ID: discover.PubkeyID(&srv.PrivateKey.PublicKey)}
	for _, p := range srv.Protocols {
		srv.ourHandshake.Caps = append(srv.ourHandshake.Caps, p.cap())
	}
	// listen/dial
	if srv.ListenAddr != "" {
		if err := srv.startListening(); err != nil {
			return err
		}
	}
	if srv.NoDial && srv.ListenAddr == "" {
		log.Warn("P2P server will be useless, neither dialing nor listening")
	}

	srv.loopWG.Add(1)
	go srv.run(dialer)
	srv.running = true
	return nil
}
```
该方法首先建立一个UDP连接，并且调用server.run启动起来，跟其它可用的节点建立连接等。关于如何建立连接，后续在讲P2P的时候，我们再来仔细的分析。到此，整个节点就启动起来了。 接下来，我们来看前面4个步骤中的第二个–创建console实例。console.go中创建一个console的New方法实现如下：
```
func New(config Config) (*Console, error) {
	// Handle unset config values gracefully
	if config.Prompter == nil {
		config.Prompter = Stdin
	}
	if config.Prompt == "" {
		config.Prompt = DefaultPrompt
	}
	if config.Printer == nil {
		config.Printer = colorable.NewColorableStdout()
	}
	// Initialize the console and return
	console := &Console{
		client:   config.Client,
		jsre:     jsre.New(config.DocRoot, config.Printer),
		prompt:   config.Prompt,
		prompter: config.Prompter,
		printer:  config.Printer,
		histPath: filepath.Join(config.DataDir, HistoryFile),
	}
	if err := console.init(config.Preload); err != nil {
		return nil, err
	}
	return console, nil
}
```
该函数首先对Config做一些默认设置，接着调用其jsre.New函数，在借这个调用init方法初始化相关RPC API。jsre.New方法的实现如下：
```
func New(assetPath string, output io.Writer) *JSRE {
	re := &JSRE{
		assetPath:     assetPath,
		output:        output,
		closed:        make(chan struct{}),
		evalQueue:     make(chan *evalReq),
		stopEventLoop: make(chan bool),
	}
	go re.runEventLoop()
	re.Set("loadScript", re.loadScript)
	re.Set("inspect", re.prettyPrintJS)
	return re
}
```
该函数创建一个JSRE的变量然后使用goroutine的方式开启一个事件监听的循环，该事件为控制台输入后，经过一些处理通过channel的方式发送过来。具体接收到的命令后续处理，请读者自己跟踪其逻辑实现。 总结下创建console的目的主要是监听控制台输入的命名。 接下来我们来看看第三个步骤「显示Welcome信息 」，该信息在通过geth进入控制台的时候，会显示一些简单的信息，我们先来看看该函数的实现：
```
func (c *Console) Welcome() {
	// Print some generic Geth metadata
	fmt.Fprintf(c.printer, "Welcome to the Geth JavaScript console!\n\n")
	c.jsre.Run(`
		console.log("instance: " + web3.version.node);
		console.log("coinbase: " + eth.coinbase);
		console.log("at block: " + eth.blockNumber + " (" + new Date(1000 * eth.getBlock(eth.blockNumber).timestamp) + ")");
		console.log(" datadir: " + admin.datadir);
	`)
	// List all the supported modules for the user to call
	if apis, err := c.client.SupportedModules(); err == nil {
		modules := make([]string, 0, len(apis))
		for api, version := range apis {
			modules = append(modules, fmt.Sprintf("%s:%s", api, version))
		}
		sort.Strings(modules)
		fmt.Fprintln(c.printer, " modules:", strings.Join(modules, " "))
	}
	fmt.Fprintln(c.printer)
}
```
该方法首先打印「Welcome to the Geth JavaScript console!」信息，这个可以直接在控制台查看到，接着使用上面创建console的jsre执行四个控制台输出写版本、区块等信息。你可以直接复制出来在命令行执行，也能看到相同的结果。下面的截图为该函数的输出信息：
[img]

接下来，我们来看最后的一个步骤「创建一个无限循环用于在控制台交互」，该函数的实现如下：
```
func (c *Console) Interactive() {
	var (
		prompt    = c.prompt          // Current prompt line (used for multi-line inputs)
		indents   = 0                 // Current number of input indents (used for multi-line inputs)
		input     = ""                // Current user input
		scheduler = make(chan string) // Channel to send the next prompt on and receive the input
	)
	// Start a goroutine to listen for promt requests and send back inputs
	go func() {
		for {
			// Read the next user input
			line, err := c.prompter.PromptInput(<-scheduler)
			if err != nil {
				// In case of an error, either clear the prompt or fail
				if err == liner.ErrPromptAborted { // ctrl-C
					prompt, indents, input = c.prompt, 0, ""
					scheduler <- ""
					continue
				}
				close(scheduler)
				return
			}
			// User input retrieved, send for interpretation and loop
			scheduler <- line
		}
	}()
	// Monitor Ctrl-C too in case the input is empty and we need to bail
	abort := make(chan os.Signal, 1)
	signal.Notify(abort, os.Interrupt)

	// Start sending prompts to the user and reading back inputs
	for {
		// Send the next prompt, triggering an input read and process the result
		scheduler <- prompt
		select {
		case <-abort:
			// User forcefully quite the console
			fmt.Fprintln(c.printer, "caught interrupt, exiting")
			return

		case line, ok := <-scheduler:
			// User input was returned by the prompter, handle special cases
			if !ok || (indents <= 0 && exit.MatchString(line)) {
				return
			}
			if onlyWhitespace.MatchString(line) {
				continue
			}
			// Append the line to the input and check for multi-line interpretation
			input += line + "\n"

			indents = countIndents(input)
			if indents <= 0 {
				prompt = c.prompt
			} else {
				prompt = strings.Repeat(".", indents*3) + " "
			}
			// If all the needed lines are present, save the command and run
			if indents <= 0 {
				if len(input) > 0 && input[0] != ' ' && !passwordRegexp.MatchString(input) {
					if command := strings.TrimSpace(input); len(c.history) == 0 || command != c.history[len(c.history)-1] {
						c.history = append(c.history, command)
						if c.prompter != nil {
							c.prompter.AppendHistory(command)
						}
					}
				}
				c.Evaluate(input)
				input = ""
			}
		}
	}
}
```
该方法首先创建一个scheduler通道，用于进程间的通信。接着使用goroutine的方式创建一个线程来监听用户在控制台的输入。接着一个无线for循环来处理从scheduler传递过来用户的输入信息。首先通过从scheduler通道读取数据line，然后做一些数据合法性验证。接着调用console的Evaluate方法将用户输入的命令交给前面的jsre处理（最终调用Do方法向evalQueue通道发送数据），最终前面初始化console时初始化的jsre用于接收用户输入命令的线程接收到数据，然后调用VM来处理，并将结果返回在控制台显示。

至此，整个以太坊就启动起来了，完成了相关AccountManager的初始化、P2P Node的启动、RPC Server启动以及控制台交互的事件监听。后续我们将分析P2P网络的启动、转账的具体实现等等。
对于分析的流程，可能有些地方有错误，麻烦大家指出，感谢~