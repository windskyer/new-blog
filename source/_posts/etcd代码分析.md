---
title: etcd代码分析
categories: Kubernetes
sage: false
date: 2021-01-06 11:16:42
tags: etcd
---

## 前提条件

etcd 版本: v3.4.6

## 服务启动入口

etcd\etcdmain\main.go

<!-- more -->
```go
package etcdmain

import (
	"fmt"
	"os"
	"strings"

	"github.com/coreos/go-systemd/daemon"
	"go.uber.org/zap"
)

func Main() {
	checkSupportArch()

	if len(os.Args) > 1 {
		cmd := os.Args[1]
		if covArgs := os.Getenv("ETCDCOV_ARGS"); len(covArgs) > 0 {
			args := strings.Split(os.Getenv("ETCDCOV_ARGS"), "\xe7\xcd")[1:]
			rootCmd.SetArgs(args)
			cmd = "grpc-proxy"
		}
		switch cmd {
		case "gateway", "grpc-proxy":
			if err := rootCmd.Execute(); err != nil {
				fmt.Fprint(os.Stderr, err)
				os.Exit(1)
			}
			return
		}
	}

	startEtcdOrProxyV2()
}

func notifySystemd(lg *zap.Logger) {
	_, err := daemon.SdNotify(false, daemon.SdNotifyReady)
	if err != nil {
		if lg != nil {
			lg.Error("failed to notify systemd for readiness", zap.Error(err))
		} else {
			plog.Errorf("failed to notify systemd for readiness: %v", err)
		}
	}
}

```

```go

// StartEtcd launches the etcd server and HTTP handlers for client/server communication.
// The returned Etcd.Server is not guaranteed to have joined the cluster. Wait
// on the Etcd.Server.ReadyNotify() channel to know when it completes and is ready for use.
func StartEtcd(inCfg *Config) (e *Etcd, err error) {
	if err = inCfg.Validate(); err != nil {
		return nil, err
	}
	serving := false
	e = &Etcd{cfg: *inCfg, stopc: make(chan struct{})}
	cfg := &e.cfg
	defer func() {
		if e == nil || err == nil {
			return
		}
		if !serving {
			// errored before starting gRPC server for serveCtx.serversC
			for _, sctx := range e.sctxs {
				close(sctx.serversC)
			}
		}
		e.Close()
		e = nil
	}()

	if e.cfg.logger != nil {
		e.cfg.logger.Info(
			"configuring peer listeners",
			zap.Strings("listen-peer-urls", e.cfg.getLPURLs()),
		)
	}
	if e.Peers, err = configurePeerListeners(cfg); err != nil {
		return e, err
	}

	if e.cfg.logger != nil {
		e.cfg.logger.Info(
			"configuring client listeners",
			zap.Strings("listen-client-urls", e.cfg.getLCURLs()),
		)
	}
	if e.sctxs, err = configureClientListeners(cfg); err != nil {
		return e, err
	}

	for _, sctx := range e.sctxs {
		e.Clients = append(e.Clients, sctx.l)
	}

	var (
		urlsmap types.URLsMap
		token   string
	)
	memberInitialized := true
	if !isMemberInitialized(cfg) {
		memberInitialized = false
		urlsmap, token, err = cfg.PeerURLsMapAndToken("etcd")
		if err != nil {
			return e, fmt.Errorf("error setting up initial cluster: %v", err)
		}
	}

	// AutoCompactionRetention defaults to "0" if not set.
	if len(cfg.AutoCompactionRetention) == 0 {
		cfg.AutoCompactionRetention = "0"
	}
	autoCompactionRetention, err := parseCompactionRetention(cfg.AutoCompactionMode, cfg.AutoCompactionRetention)
	if err != nil {
		return e, err
	}

	backendFreelistType := parseBackendFreelistType(cfg.ExperimentalBackendFreelistType)

	srvcfg := etcdserver.ServerConfig{
		Name:                       cfg.Name,
		ClientURLs:                 cfg.ACUrls,
		PeerURLs:                   cfg.APUrls,
		DataDir:                    cfg.Dir,
		DedicatedWALDir:            cfg.WalDir,
		SnapshotCount:              cfg.SnapshotCount,
		SnapshotCatchUpEntries:     cfg.SnapshotCatchUpEntries,
		MaxSnapFiles:               cfg.MaxSnapFiles,
		MaxWALFiles:                cfg.MaxWalFiles,
		InitialPeerURLsMap:         urlsmap,
		InitialClusterToken:        token,
		DiscoveryURL:               cfg.Durl,
		DiscoveryProxy:             cfg.Dproxy,
		NewCluster:                 cfg.IsNewCluster(),
		PeerTLSInfo:                cfg.PeerTLSInfo,
		TickMs:                     cfg.TickMs,
		ElectionTicks:              cfg.ElectionTicks(),
		InitialElectionTickAdvance: cfg.InitialElectionTickAdvance,
		AutoCompactionRetention:    autoCompactionRetention,
		AutoCompactionMode:         cfg.AutoCompactionMode,
		QuotaBackendBytes:          cfg.QuotaBackendBytes,
		BackendBatchLimit:          cfg.BackendBatchLimit,
		BackendFreelistType:        backendFreelistType,
		BackendBatchInterval:       cfg.BackendBatchInterval,
		MaxTxnOps:                  cfg.MaxTxnOps,
		MaxRequestBytes:            cfg.MaxRequestBytes,
		StrictReconfigCheck:        cfg.StrictReconfigCheck,
		ClientCertAuthEnabled:      cfg.ClientTLSInfo.ClientCertAuth,
		AuthToken:                  cfg.AuthToken,
		BcryptCost:                 cfg.BcryptCost,
		CORS:                       cfg.CORS,
		HostWhitelist:              cfg.HostWhitelist,
		InitialCorruptCheck:        cfg.ExperimentalInitialCorruptCheck,
		CorruptCheckTime:           cfg.ExperimentalCorruptCheckTime,
		PreVote:                    cfg.PreVote,
		Logger:                     cfg.logger,
		LoggerConfig:               cfg.loggerConfig,
		LoggerCore:                 cfg.loggerCore,
		LoggerWriteSyncer:          cfg.loggerWriteSyncer,
		Debug:                      cfg.Debug,
		ForceNewCluster:            cfg.ForceNewCluster,
		EnableGRPCGateway:          cfg.EnableGRPCGateway,
		EnableLeaseCheckpoint:      cfg.ExperimentalEnableLeaseCheckpoint,
		CompactionBatchLimit:       cfg.ExperimentalCompactionBatchLimit,
	}
	print(e.cfg.logger, *cfg, srvcfg, memberInitialized)
	if e.Server, err = etcdserver.NewServer(srvcfg); err != nil {
		return e, err
	}

	// buffer channel so goroutines on closed connections won't wait forever
	e.errc = make(chan error, len(e.Peers)+len(e.Clients)+2*len(e.sctxs))

	// newly started member ("memberInitialized==false")
	// does not need corruption check
	if memberInitialized {
		if err = e.Server.CheckInitialHashKV(); err != nil {
			// set "EtcdServer" to nil, so that it does not block on "EtcdServer.Close()"
			// (nothing to close since rafthttp transports have not been started)
			e.Server = nil
			return e, err
		}
	}
	e.Server.Start()

	if err = e.servePeers(); err != nil {
		return e, err
	}
	if err = e.serveClients(); err != nil {
		return e, err
	}
	if err = e.serveMetrics(); err != nil {
		return e, err
	}

	if e.cfg.logger != nil {
		e.cfg.logger.Info(
			"now serving peer/client/metrics",
			zap.String("local-member-id", e.Server.ID().String()),
			zap.Strings("initial-advertise-peer-urls", e.cfg.getAPURLs()),
			zap.Strings("listen-peer-urls", e.cfg.getLPURLs()),
			zap.Strings("advertise-client-urls", e.cfg.getACURLs()),
			zap.Strings("listen-client-urls", e.cfg.getLCURLs()),
			zap.Strings("listen-metrics-urls", e.cfg.getMetricsURLs()),
		)
	}
	serving = true
	return e, nil
}
```


```go
// 加载本地snap 和 wal
case haveWAL:
		if err = fileutil.IsDirWriteable(cfg.MemberDir()); err != nil {
			return nil, fmt.Errorf("cannot write to member directory: %v", err)
		}

		if err = fileutil.IsDirWriteable(cfg.WALDir()); err != nil {
			return nil, fmt.Errorf("cannot write to WAL directory: %v", err)
		}

		if cfg.ShouldDiscover() {
			if cfg.Logger != nil {
				cfg.Logger.Warn(
					"discovery token is ignored since cluster already initialized; valid logs are found",
					zap.String("wal-dir", cfg.WALDir()),
				)
			} else {
				plog.Warningf("discovery token ignored since a cluster has already been initialized. Valid log found at %q", cfg.WALDir())
			}
		}
		snapshot, err = ss.Load()
		if err != nil && err != snap.ErrNoSnapshot {
			return nil, err
		}
		if snapshot != nil {
			if err = st.Recovery(snapshot.Data); err != nil {
				if cfg.Logger != nil {
					cfg.Logger.Panic("failed to recover from snapshot")
				} else {
					plog.Panicf("recovered store from snapshot error: %v", err)
				}
			}

			if cfg.Logger != nil {
				cfg.Logger.Info(
					"recovered v2 store from snapshot",
					zap.Uint64("snapshot-index", snapshot.Metadata.Index),
					zap.String("snapshot-size", humanize.Bytes(uint64(snapshot.Size()))),
				)
			} else {
				plog.Infof("recovered store from snapshot at index %d", snapshot.Metadata.Index)
			}

			if be, err = recoverSnapshotBackend(cfg, be, *snapshot); err != nil {
				if cfg.Logger != nil {
					cfg.Logger.Panic("failed to recover v3 backend from snapshot", zap.Error(err))
				} else {
					plog.Panicf("recovering backend from snapshot error: %v", err)
				}
			}
			if cfg.Logger != nil {
				s1, s2 := be.Size(), be.SizeInUse()
				cfg.Logger.Info(
					"recovered v3 backend from snapshot",
					zap.Int64("backend-size-bytes", s1),
					zap.String("backend-size", humanize.Bytes(uint64(s1))),
					zap.Int64("backend-size-in-use-bytes", s2),
					zap.String("backend-size-in-use", humanize.Bytes(uint64(s2))),
				)
			}
		}

		if !cfg.ForceNewCluster {
			id, cl, n, s, w = restartNode(cfg, snapshot)
		} else {
			id, cl, n, s, w = restartAsStandaloneNode(cfg, snapshot)
		}

		cl.SetStore(st)
		cl.SetBackend(be)
		cl.Recover(api.UpdateCapability)
		if cl.Version() != nil && !cl.Version().LessThan(semver.Version{Major: 3}) && !beExist {
			os.RemoveAll(bepath)
			return nil, fmt.Errorf("database file (%v) of the backend is missing", bepath)
		}

```