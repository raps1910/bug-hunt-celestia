# bug-hunt-celestia



Skip to content
Pull requests
Issues
Marketplace
Explore
@Kuzmin55
celestiaorg /
celestia-node
Public

Code
Issues 130
Pull requests 11
Actions
Projects 1
Wiki
Security

    Insights

(cmd | node/rpc): Make RPC port configurable and ensure port 0 used f…

…or swamp (#617)

    main (#617) 

@renaynay
renaynay committed 24 days ago
1 parent 8dc6e20 commit f6b97b9c13ba5d4f20ba3fbb4721909b3fc49d7c
Showing
with 117 additions and 20 deletions.
7
cmd/celestia/bridge.go
@@ -27,13 +27,15 @@ func init() {
			cmdnode.CoreFlags(),
			cmdnode.TrustedHashFlags(),
			cmdnode.MiscFlags(),
			cmdnode.RPCFlags(),
		),
		cmdnode.Start(
			cmdnode.NodeFlags(node.Bridge),
			cmdnode.P2PFlags(),
			cmdnode.CoreFlags(),
			cmdnode.TrustedHashFlags(),
			cmdnode.MiscFlags(),
			cmdnode.RPCFlags(),
		),
		bridgeKeyCmd,
	)
@@ -76,6 +78,11 @@ var bridgeCmd = &cobra.Command{
			return err
		}

		err = cmdnode.ParseRPCFlags(cmd, env)
		if err != nil {
			return err
		}

		return nil
	},
}
7
cmd/celestia/full.go
@@ -29,6 +29,7 @@ func init() {
			// NOTE: for now, state-related queries can only be accessed
			// over an RPC connection with a celestia-core node.
			cmdnode.CoreFlags(),
			cmdnode.RPCFlags(),
		),
		cmdnode.Start(
			cmdnode.NodeFlags(node.Full),
@@ -38,6 +39,7 @@ func init() {
			// NOTE: for now, state-related queries can only be accessed
			// over an RPC connection with a celestia-core node.
			cmdnode.CoreFlags(),
			cmdnode.RPCFlags(),
		),
		fullKeyCmd,
	)
@@ -80,6 +82,11 @@ var fullCmd = &cobra.Command{
			return err
		}

		err = cmdnode.ParseRPCFlags(cmd, env)
		if err != nil {
			return err
		}

		return nil
	},
}
7
cmd/celestia/light.go
@@ -29,6 +29,7 @@ func init() {
			// NOTE: for now, state-related queries can only be accessed
			// over an RPC connection with a celestia-core node.
			cmdnode.CoreFlags(),
			cmdnode.RPCFlags(),
		),
		cmdnode.Start(
			cmdnode.NodeFlags(node.Light),
@@ -38,6 +39,7 @@ func init() {
			// NOTE: for now, state-related queries can only be accessed
			// over an RPC connection with a celestia-core node.
			cmdnode.CoreFlags(),
			cmdnode.RPCFlags(),
		),
		lightKeyCmd,
	)
@@ -79,6 +81,11 @@ var lightCmd = &cobra.Command{
			return err
		}

		err = cmdnode.ParseRPCFlags(cmd, env)
		if err != nil {
			return err
		}

		return nil
	},
}
44
cmd/flags_rpc.go
@@ -0,0 +1,44 @@
package cmd

import (
	"github.com/spf13/cobra"
	flag "github.com/spf13/pflag"

	"github.com/celestiaorg/celestia-node/node"
)

var (
	addrFlag = "rpc.addr"
	portFlag = "rpc.port"
)

// RPCFlags gives a set of hardcoded node/rpc package flags.
func RPCFlags() *flag.FlagSet {
	flags := &flag.FlagSet{}

	flags.String(
		addrFlag,
		"",
		"Set a custom RPC listen address (default: localhost)",
	)
	flags.String(
		portFlag,
		"",
		"Set a custom RPC port (default: 26658)",
	)

	return flags
}

// ParseRPCFlags parses RPC flags from the given cmd and applies values to Env.
func ParseRPCFlags(cmd *cobra.Command, env *Env) error {
	addr := cmd.Flag(addrFlag).Value.String()
	if addr != "" {
		env.AddOptions(node.WithRPCAddress(addr))
	}
	port := cmd.Flag(portFlag).Value.String()
	if port != "" {
		env.AddOptions(node.WithRPCPort(port))
	}
	return nil
}
16
node/components.go
@@ -10,9 +10,7 @@ import (
	"go.uber.org/fx"

	"github.com/celestiaorg/celestia-node/libs/fxutil"

	nodecore "github.com/celestiaorg/celestia-node/node/core"

	"github.com/celestiaorg/celestia-node/node/p2p"
	"github.com/celestiaorg/celestia-node/node/rpc"
	"github.com/celestiaorg/celestia-node/node/services"
@@ -74,18 +72,8 @@ func baseComponents(cfg *Config, store Store) fx.Option {
		// state components
		fx.Provide(statecomponents.NewService),
		fx.Provide(statecomponents.CoreAccessor(cfg.Core.GRPCAddr)),
		// RPC component
		fx.Provide(func(lc fx.Lifecycle) *rpc.Server {
			// TODO @renaynay @Wondertan: not providing any custom config
			//  functionality here as this component is meant to be removed on
			//  implementation of https://github.com/celestiaorg/celestia-node/pull/506.
			serv := rpc.NewServer(rpc.DefaultConfig())
			lc.Append(fx.Hook{
				OnStart: serv.Start,
				OnStop:  serv.Stop,
			})
			return serv
		}),
		// RPCServer component
		fx.Provide(rpc.ServerComponent(cfg.RPC)),
	)
}

2
node/config.go
@@ -37,11 +37,13 @@ func DefaultConfig(tp Type) *Config {
		}
	case Light:
		return &Config{
			RPC:      rpc.DefaultConfig(),
			P2P:      p2p.DefaultConfig(),
			Services: services.DefaultConfig(),
		}
	case Full:
		return &Config{
			RPC:      rpc.DefaultConfig(),
			P2P:      p2p.DefaultConfig(),
			Services: services.DefaultConfig(),
		}
16
node/config_opts.go
@@ -16,6 +16,22 @@ func WithGRPCEndpoint(address string) Option {
	}
}

// WithRPCPort configures Node to expose the given port for RPC
// queries.
func WithRPCPort(port string) Option {
	return func(sets *settings) {
		sets.cfg.RPC.Port = port
	}
}

// WithRPCAddress configures Node to listen on the given address for RPC
// queries.
func WithRPCAddress(addr string) Option {
	return func(sets *settings) {
		sets.cfg.RPC.Address = addr
	}
}

// WithTrustedHash sets TrustedHash to the Config.
func WithTrustedHash(hash string) Option {
	return func(sets *settings) {
2
node/node.go
@@ -99,7 +99,7 @@ func (n *Node) Start(ctx context.Context) error {
	}

	// TODO(@Wondertan): Print useful information about the node:
	//  * API/RPC address
	//  * API/RPCServer address
	log.Infof("started %s Node", n.Type)

	addrs, err := peer.AddrInfoToP2pAddrs(host.InfoFromHost(n.Host))
1
node/node_full_test.go
@@ -16,7 +16,6 @@ func TestNewFullAndLifecycle(t *testing.T) {
	assert.NotZero(t, node.Type)

	startCtx, cancel := context.WithCancel(context.Background())
	defer cancel()

	err := node.Start(startCtx)
	require.NoError(t, err)
19
node/rpc/component.go
@@ -0,0 +1,19 @@
package rpc

import (
	"go.uber.org/fx"
)

// ServerComponent constructs a new RPC Server from the given Config.
// TODO @renaynay @Wondertan: this component is meant to be removed on implementation
//  of https://github.com/celestiaorg/celestia-node/pull/506.
func ServerComponent(cfg Config) func(lc fx.Lifecycle) *Server {
	return func(lc fx.Lifecycle) *Server {
		serv := NewServer(cfg)
		lc.Append(fx.Hook{
			OnStart: serv.Start,
			OnStop:  serv.Stop,
		})
		return serv
	}
}
6
node/rpc/config.go
@@ -1,12 +1,14 @@
package rpc

type Config struct {
	ListenAddr string
	Address string
	Port    string
}

func DefaultConfig() Config {
	return Config{
		Address: "0.0.0.0",
		// do NOT expose the same port as celestia-core by default so that both can run on the same machine
		ListenAddr: "0.0.0.0:26658",
		Port: "26658",
	}
}
4
node/rpc/server.go
@@ -2,6 +2,7 @@ package rpc

import (
	"context"
	"fmt"
	"net"
	"net/http"

@@ -36,7 +37,8 @@ func NewServer(cfg Config) *Server {

// Start starts the RPC Server, listening on the given address.
func (s *Server) Start(context.Context) error {
	listener, err := net.Listen("tcp", s.cfg.ListenAddr)
	listenAddr := fmt.Sprintf("%s:%s", s.cfg.Address, s.cfg.Port)
	listener, err := net.Listen("tcp", listenAddr)
	if err != nil {
		return err
	}
5
node/rpc/server_test.go
@@ -12,7 +12,10 @@ import (
)

func TestServer(t *testing.T) {
	server := NewServer(DefaultConfig())
	server := NewServer(Config{
		Address: "0.0.0.0",
		Port:    "0",
	})

	ctx, cancel := context.WithCancel(context.Background())
	t.Cleanup(cancel)
1
node/tests/swamp/swamp.go
@@ -249,6 +249,7 @@ func (s *Swamp) newNode(t node.Type, store node.Store, options ...node.Option) *
		node.WithHost(s.createPeer(ks)),
		node.WithTrustedHash(s.trustedHash),
		node.WithNetwork(params.Private),
		node.WithRPCPort("0"),
	)

	node, err := node.New(t, store, options...)
0 comments on commit f6b97b9

Attach files by dragging & dropping, selecting or pasting them.

You’re not receiving notifications from this thread.

    © 2022 GitHub, Inc.

    Terms
    Privacy
    Security
    Status
    Docs
    Contact GitHub
    Pricing
    API
    Training
    Blog
    About

Loading complete
