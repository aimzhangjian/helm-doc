# 理解Helm安装升级参数wait、wait-for-jobs、atomic、cleanup-on-fail、timeout
Helm执行install、upgrade指令时可以指定--wait、--atomic、--timeout参数，并且upgrade指令还可以指定--cleanup-on-fail参数。此篇文章将从
helm源码的角度去理解这几个参数的实现及其意义，并给出如何自己管理chart应用的状态

## wait、wait-for-jobs参数

### 语义
Helm应用在执行升级安装时，其实现逻辑是将values参数及内置参数按模版渲染成K8s资源，并发送给K8s集群。当K8s集群接收资源后helm就认为安装成功，
此时应用的状态将更新成deployed。此时其实K8s并没有真正将Chart包中定义的资源创建、更新好，如果K8s资源更新创建失败，整个发布其实不能算是成功了，
因为根本没有运行起来。而添加了--wait参数后，helm将启动一个循环，查看Chart包下所有pod资源在K8s中是否正常更新创建并运行。如果所有pod资源都正常被K8s
更新创建并运行，则将发布状态设置为deployed；如果有pod资源出现异常将返回failed状态。如果到达timeout设置的超时时间仍然所有pod资源没有正常运行，则
返回failed状态。wait-for-jobs参数与wait参数类似，但需要先开启wait参数，否则不生效，wait-for-jobs区别于wait参数的地方是需要等到所有job完成。
由此可见在设置--wait参数后，如果升级安装操作正常完成，返回deployed状态，则可以确认此次升级安装是执行成功了。但我们也发现有一个不太完美的事情，
由于要设置超时时间，如果超过超时时间则认为这次安装失败，有可能由于设置的超时时间不太合理导致虚假故障。并且这个循环操作是在helm中执行的，
如果是自己依赖helm源码做的客户化工具，那么客户化工具重启或升级也会加大出问题的风险。对于哪些在升级安装前需要通过钩子实现初始化种子数据的
Chart而言超时的可能性更大，因为初始化种子数据一般耗时较长，而且在3.2.0之前是串行执行，在3.2.0之后对于相同权重的钩子资源执行顺序与非钩子资源相同

### helm客户端中如何设置及源码处理逻辑

#### 通过helm升级、安装Chart并开启wait、wait-for-jobs
我们需要添加helm客户端的依赖。下面列出的helm依赖是我自己客户化的helm客户端

```
require (
	github.com/aimjianzhang/helm v0.0.0-20220411123222-8d7d0eade5d5
)
```
我们需要自己写安装chart包的逻辑，这个安装方法可以参考helm客户端中的helm.cmd.helm.install.go@runInstall方法。通过设置创建的Install对象的
Wait属性为ture可以启用wait,我们认真观察一下，可以发现Install对象的属性其实和我们通过helm命令install安装时可以指定的参数是一样的，helm的
install命令其实最后转换成的就是Install对象。[获取代码](https://github.com/aimjianzhang/helm-agent)

```
var helmInstallReleaseParams model.HelmInstallReleaseParams
	err := json.Unmarshal([]byte(cmd.Payload), helmInstallReleaseParams)
	if err != nil {
		return nil, NewResponseReleaseError(cmd.Key, cmd.Type, err.Error())
	}

	chartPathOptions := action.ChartPathOptions{
		RepoURL:  helmInstallReleaseParams.RepoURL,
		Version:  helmInstallReleaseParams.ChartVersion,
		Username: helmInstallReleaseParams.RepoUserName,
		Password: helmInstallReleaseParams.RepoPassword,
	}
	// 获取helm配置信息
	actionConfiguration, settings := getCfg(helmInstallReleaseParams.Namespace)

	// 创建install客户端，可以设置helm install支持的参数
	instClient := action.NewInstall(actionConfiguration)
	// 设置chart仓库地址及版本、认证信息等
	instClient.ChartPathOptions = chartPathOptions
	// 设置安装chart包等待所有pod起好
	instClient.Wait = true
	// 设置安装chart包等待所有job运行完成
	instClient.WaitForJobs = true
	// 设置安装是原子的，如果安装失败会回滚到上一个正常版本
	instClient.Atomic = true
	// 需要在那个命名空间下安装
	instClient.Namespace = helmInstallReleaseParams.Namespace
	// release的名称
	instClient.ReleaseName = helmInstallReleaseParams.ReleaseName

	// 查找chart返回完整路径或错误。如果有配置验证chart，将尝试验证
	cp, err := instClient.ChartPathOptions.LocateChart(helmInstallReleaseParams.ChartName, settings)
	if err != nil {
		return nil, NewResponseReleaseError(cmd.Key, cmd.Type, err.Error())
	}

	valuesOptions := getValuesOptions(helmInstallReleaseParams.Values)
	getters := getter.All(settings)
	vals, err := valuesOptions.MergeValues(getters)
	if err != nil {
		return nil, NewResponseReleaseError(cmd.Key, cmd.Type, err.Error())
	}
	// Check chart dependencies to make sure all are present in /charts
	chartRequested, err := loader.Load(cp)
	if err != nil {
		return nil, NewResponseReleaseError(cmd.Key, cmd.Type, err.Error())
	}
	if err = checkIfInstallable(chartRequested); err != nil {
		return nil, NewResponseReleaseError(cmd.Key, cmd.Type, err.Error())
	}
	if chartRequested.Metadata.Deprecated {
		glog.Info("This chart is deprecated")
	}
	if req := chartRequested.Metadata.Dependencies; req != nil {
		if err = action.CheckDependencies(chartRequested, req); err != nil {
			if instClient.DependencyUpdate {
				man := &downloader.Manager{
					ChartPath:        cp,
					Keyring:          instClient.ChartPathOptions.Keyring,
					SkipUpdate:       false,
					Getters:          getters,
					RepositoryConfig: settings.RepositoryConfig,
					RepositoryCache:  settings.RepositoryCache,
					Debug:            settings.Debug,
				}
				if err = man.Update(); err != nil {
					return nil, NewResponseReleaseError(cmd.Key, cmd.Type, err.Error())
				}
				if chartRequested, err = loader.Load(cp); err != nil {
					return nil, NewResponseReleaseError(cmd.Key, cmd.Type, err.Error())
				}
			} else {
				return nil, NewResponseReleaseError(cmd.Key, cmd.Type, err.Error())
			}
		}
	}
	responseRelease, err := instClient.Run(chartRequested, vals)
	if err != nil {
		return nil, NewResponseReleaseError(cmd.Key, cmd.Type, err.Error())
	}
	responseReleaseJson, err := json.Marshal(responseRelease)
	if err != nil {
		return nil, NewResponseReleaseError(cmd.Key, cmd.Type, err.Error())
	}
	return nil, &model.Packet{
		Key:     cmd.Key,
		Type:    cmd.Type,
		Payload: string(responseReleaseJson),
}
```

#### 针对wait、wait-for-jobs参数helm做了那些处理

1. 判断是否开启wait-for-jobs配置，如果开启执行helm.pkg.kube.interface@WaitWithJobs方法，等待所有pod资源及job资源执行成功；如果只开启
wait参数，则执行helm.pkg.kube.interface@Wait，等待所有pod资源执行成功
```
if i.Wait {
    if i.WaitForJobs {
        if err := i.cfg.KubeClient.WaitWithJobs(resources, i.Timeout); err != nil {
            i.reportToRun(c, rel, err)
            return
        }
    } else {
        if err := i.cfg.KubeClient.Wait(resources, i.Timeout); err != nil {
            i.reportToRun(c, rel, err)
            return
        }
    }
}
```
2. helm.pkg.kube.interface@Wait方法需要传入部署的资源对象列表，但此处的资源对象列表不包括钩子资源，如果需要包括钩子资源需要修改源码。
还需要传入超时时间，此超时时间可以在Install对象中通过Timeout参数，当然通过helm install命令指定--timeout参数值，也是这里所说的超时时间。
此方法会在超时时间内轮询查看资源对象列表中的所有pod资源是否创建成功并到达就绪状态。如果在超时时间内，所有pod资源都到达就绪状态，则执行成功，
否则安装失败给出具体原因。下面我们具体分析一下这个方法
   
```
func (c *Client) Wait(resources ResourceList, timeout time.Duration) error {
	cs, err := c.getKubeClient()
	if err != nil {
		return err
	}
	checker := NewReadyChecker(cs, c.Log, PausedAsReady(true))
	w := waiter{
		c:       checker,
		log:     c.Log,
		timeout: timeout,
	}
	return w.waitForResources(resources)
}
```

- 首先获取kubectl客户端

- 创建新的检查器，初始化client、log、 pausedAsReady属性值,注意这里并没有初始化checkJobs属性
    ```
    func NewReadyChecker(cl kubernetes.Interface, log func(string, ...interface{}), opts ...ReadyCheckerOption) ReadyChecker {
        c := ReadyChecker{
            client: cl,
            log:    log,
        }
        if c.log == nil {
            c.log = nopLogger
        }
        for _, opt := range opts {
            opt(&c)
        }
        return c
    }
    ```
- 创建waiter对象并调用waitForResources等待资源达到预期效果

waitForResources方法通过轮询检查所有pod、PVC、Service、Job是否在超时时间内正常运行起来。我们看一下此方法是如何实现的
```
func (w *waiter) waitForResources(created ResourceList) error {
	w.log("beginning wait for %d resources with timeout of %v", len(created), w.timeout)

	ctx, cancel := context.WithTimeout(context.Background(), w.timeout)
	defer cancel()

	return wait.PollImmediateUntil(2*time.Second, func() (bool, error) {
		for _, v := range created {
			ready, err := w.c.IsReady(ctx, v)
			if !ready || err != nil {
				return false, err
			}
		}
		return true, nil
	}, ctx.Done())
}
```

- 首先调用context.WithTimeout方法传入Context和超时时间，返回子Context和取消函数cancel。监听子Context的Done通道，当调用其cancel函数时，
Context的Done通道会收到消息退出。当到达超时时间时会自动调用cancel方法
  
- 方法执行结束后执行cancel方法退出

- wait.PollImmediateUntil方法接收三个参数，第一个参数指定轮询时间，这里写死每隔两秒执行一次检查操作，第二个参数ConditionFunc就是要执行的
检查操作，第三个参数是第一步创建的子Context的Done通道，当cancel方法触发时将收到消息
  
- 我们看一下wait.PollImmediateUntil方法的第二个参数即检查操作，检查操作循环资源列表调用w.c.IsReady方法检查资源是否处于就绪状态及是否出错，
如果自己做K8s管理功能，可以参考这个方法看就绪状态是如何判定的。如果没有处于就绪状态或者有异常信息，则直接返回false及错误信息。如果没有错误信息并且
返回的是false，则继续循环检查，如果有错误信息，则结束检查返回错误信息。如果所有资源都已就绪则返回true，也会停止检查返回安装成功
  
接下来我们学习一下wait.PollImmediateUntil方法是如何定期检查的
```
func PollImmediateUntil(interval time.Duration, condition ConditionFunc, stopCh <-chan struct{}) error {
	ctx, cancel := contextForChannel(stopCh)
	defer cancel()
	return PollImmediateUntilWithContext(ctx, interval, condition.WithContext())
}
```

- 首先调用contextForChannel传入停止通道。完成对停止通道的监听，以及返回新的子context及cancel取消函数
  ```
    func contextForChannel(parentCh <-chan struct{}) (context.Context, context.CancelFunc) {
      ctx, cancel := context.WithCancel(context.Background())
  
      go func() {
          select {
          case <-parentCh:
              cancel()
          case <-ctx.Done():
          }
      }()
      return ctx, cancel
    }
  ```
  - 调用context.WithCancel方法获取子context及取消函数
  
  - 并发执行select操作，监听传入的外出停止通道。当外层停止通道收到取消消息，将调用上步获取的取消函数，退出监听
  
  - 返回创建的子context及取消函数
  
- 方法结束调用cancel取消函数

- PollImmediateUntilWithContext方法将定时调用condition条件进行检查
  ```
    func PollImmediateUntilWithContext(ctx context.Context, interval time.Duration, condition ConditionWithContextFunc) error {
      return poll(ctx, true, poller(interval, 0), condition)
    }
  ```
  
  - 此方法调用了poll未公开方法及poller方法。poller方法将产生定时通道，poll将获取并监听定时通道并执行condition方法
  

poll方法接收四个参数。第一个参数是ctx context.Context，此参数可以接收Done通道消息，当cancel方法调用时将向Done通道发送消息；第二个参数
immediate bool，此参数控制是否要立即调用condition方法检查；第三个参数wait WaitWithContextFunc，调用此参数返回一个通道，监听该返回通道
将定时调用处理逻辑；第四个参数condition ConditionWithContextFunc，该参数就是检查逻辑的方法
```
func poll(ctx context.Context, immediate bool, wait WaitWithContextFunc, condition ConditionWithContextFunc) error {
	if immediate {
		done, err := runConditionWithCrashProtectionWithContext(ctx, condition)
		if err != nil {
			return err
		}
		if done {
			return nil
		}
	}

	select {
	case <-ctx.Done():
		// returning ctx.Err() will break backward compatibility
		return ErrWaitTimeout
	default:
		return WaitForWithContext(ctx, wait, condition)
	}
}
```

- immediate为true，则直接调用runConditionWithCrashProtectionWithContext方法调用condition检查逻辑，返回done和错误信息。当有错误消息
时直接返回错误消息，安装操作失败；当done为true，则表示安装成功
  
- 监听ctx.Done()的done通道，如果接到消息则表明到达超时时间，返回超时异常

- 默认执行WaitForWithContext方法
  
  ```
    func WaitForWithContext(ctx context.Context, wait WaitWithContextFunc, fn ConditionWithContextFunc) error {
      waitCtx, cancel := context.WithCancel(context.Background())
      defer cancel()
      c := wait(waitCtx)
      for {
          select {
          case _, open := <-c:
              ok, err := runConditionWithCrashProtectionWithContext(ctx, fn)
              if err != nil {
                  return err
              }
              if ok {
                  return nil
              }
              if !open {
                  return ErrWaitTimeout
              }
          case <-ctx.Done():
              // returning ctx.Err() will break backward compatibility
              return ErrWaitTimeout
          }
      }
   }
  ```

  - 获取子context和cancel函数
  
  - 调用wait方法传入子context，并返回通道。在wait方法中将监听子context的Done通道，当接收到消息就退出
  
  - 循环监听获取的通道，每隔2秒将接收道通道消息，调用检查逻辑。返回ok及err，当有err信息，则返回错误信息；当ok为true，则直接返回，表示安装
  成功，如果通道关闭了，则表明超时，返回超时错误
    
  - 循环监听ctx的Done通道，当接收到消息，则表明超时，返回超时错误信息

上面我们说到wait方法返回了一个通道，并每隔2秒向通道发送消息。我们下面看一下调用的wait方法的具体逻辑

```
func poller(interval, timeout time.Duration) WaitWithContextFunc {
	return WaitWithContextFunc(func(ctx context.Context) <-chan struct{} {
		ch := make(chan struct{})

		go func() {
			defer close(ch)

			tick := time.NewTicker(interval)
			defer tick.Stop()

			var after <-chan time.Time
			if timeout != 0 {
				// time.After is more convenient, but it
				// potentially leaves timers around much longer
				// than necessary if we exit early.
				timer := time.NewTimer(timeout)
				after = timer.C
				defer timer.Stop()
			}

			for {
				select {
				case <-tick.C:
					// If the consumer isn't ready for this signal drop it and
					// check the other channels.
					select {
					case ch <- struct{}{}:
					default:
					}
				case <-after:
					return
				case <-ctx.Done():
					return
				}
			}
		}()

		return ch
	})
}
```
- 首先创建了一个通道ch

- 并发执行匿名方法中的逻辑，在方法结束时执行close方法关闭ch

- 调用time.NewTicker(interval)方法创建一个定时器tick，每隔2秒执行一次

- 在方法执行结束时停止定时器

- 如果设置超时时间，则设置一个定时器after

- 循环监听tick每隔2秒会往ch通道发送消息

- 循环监听after，到达超时时间将直接停止监听，如果没有设置超时时间将永远不会触发

- 循环监听ctx的Done通道，接收到消息将直接退出

## atomic参数

### 语言
如果开启atomic设置，则自动会开启wait参数。在安装、升级中atomic参数有不同的逻辑。在安装过程中，如果安装失败，atomic会删除失败安装；在升级
过程中，如果升级失败，atomic会回滚升级失败所做的变更

### helm客户端中如何设置及源码处理逻辑

#### 如何设置
设置过程与上述wait参数设置过程相似

#### 安装过程源码解析
在安装过程中，首先会判断是否开启atomic，如果开启会自动将wait设置为true
```
i.Wait = i.Wait || i.Atomic // 根据Atomic是否开启自动设置Wait参数
```

atomic的处理逻辑在helm.pkg.action.install.failRelease方法中。如果开启atomic会调用uninstall删除失败的安装
```
func (i *Install) failRelease(rel *release.Release, err error) (*release.Release, error) {
  rel.SetStatus(release.StatusFailed, fmt.Sprintf("Release %q failed: %s", i.ReleaseName, err.Error()))
  if i.Atomic {
      i.cfg.Log("Install failed and atomic is set, uninstalling release")
      uninstall := NewUninstall(i.cfg)
      uninstall.DisableHooks = i.DisableHooks
      uninstall.KeepHistory = false
      uninstall.Timeout = i.Timeout
      if _, uninstallErr := uninstall.Run(i.ReleaseName); uninstallErr != nil {
          return rel, errors.Wrapf(uninstallErr, "an error occurred while uninstalling the release. original install error: %s", err)
      }
      return rel, errors.Wrapf(err, "release %s failed, and has been uninstalled due to atomic being set", i.ReleaseName)
  }
  i.recordRelease(rel) // Ignore the error, since we have another error to deal with.
  return rel, err
}
```
#### 升级过程源码解析
首先判断是否开启atomic，如果开启会自动将wait设置为true
```
u.Wait = u.Wait || u.Atomic
```

atomic的处理逻辑在helm.pkg.action.upgrade.failRelease方法中。如果开启atomic会先查询历史安装、更新列表，找到最近安装成功的历史记录，并
回滚安装，这里的回滚是一次新的部署。如果历史记录没有成功的安装记录，直接抛出异常
```
if u.Atomic {
      u.cfg.Log("Upgrade failed and atomic is set, rolling back to last successful release")

      // As a protection, get the last successful release before rollback.
      // If there are no successful releases, bail out
      hist := NewHistory(u.cfg)
      fullHistory, herr := hist.Run(rel.Name)
      if herr != nil {
          return rel, errors.Wrapf(herr, "an error occurred while finding last successful release. original upgrade error: %s", err)
      }

      // There isn't a way to tell if a previous release was successful, but
      // generally failed releases do not get superseded unless the next
      // release is successful, so this should be relatively safe
      filteredHistory := releaseutil.FilterFunc(func(r *release.Release) bool {
          return r.Info.Status == release.StatusSuperseded || r.Info.Status == release.StatusDeployed
      }).Filter(fullHistory)
      if len(filteredHistory) == 0 {
          return rel, errors.Wrap(err, "unable to find a previously successful release when attempting to rollback. original upgrade error")
      }

      releaseutil.Reverse(filteredHistory, releaseutil.SortByRevision)

      rollin := NewRollback(u.cfg)
      rollin.Version = filteredHistory[0].Version
      rollin.Wait = true
      rollin.WaitForJobs = u.WaitForJobs
      rollin.DisableHooks = u.DisableHooks
      rollin.Recreate = u.Recreate
      rollin.Force = u.Force
      rollin.Timeout = u.Timeout
      if rollErr := rollin.Run(rel.Name); rollErr != nil {
          return rel, errors.Wrapf(rollErr, "an error occurred while rolling back the release. original upgrade error: %s", err)
      }
      return rel, errors.Wrapf(err, "release %s failed, and has been rolled back due to atomic being set", rel.Name)
  }
```

## cleanup-on-fail参数
此参数在更新应用时可设置，在安装应用时是没有此参数的。开启此参数将在安装失败后删除新增的资源列表，更新应用时将资源分为了新增需要创建的资源、
有变化的需要更新资源、需要被删除的资源
```
if u.CleanupOnFail && len(created) > 0 {
  u.cfg.Log("Cleanup on fail set, cleaning up %d resources", len(created))
  _, errs := u.cfg.KubeClient.Delete(created)
  if errs != nil {
      var errorList []string
      for _, e := range errs {
          errorList = append(errorList, e.Error())
      }
      return rel, errors.Wrapf(fmt.Errorf("unable to cleanup resources: %s", strings.Join(errorList, ", ")), "an error occurred while cleaning up resources. original upgrade error: %s", err)
  }
  u.cfg.Log("Resource cleanup complete")
}
```



     




