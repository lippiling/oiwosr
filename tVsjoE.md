久彝嘏狄圆


Linux selinux_ops挂载点安全钩子与sb_alloc_security

SELinux通过selinux_hooks安全框架结构体注册了全套文件系统超级块(superblock)操作钩子。这些钩子控制挂载点的安全上下文分配、初始化、释放以及文件系统类型的策略标记。selinux_sb_alloc_security是挂载路径上的第一个SELinux钩子，在所有文件系统超级块分配时被调用。

selinux_sb_alloc_security的实现：

```c
static int selinux_sb_alloc_security(struct super_block *sb)
{
    struct superblock_security_struct *sbsec;

    /* 从SELinux专用的slab缓存分配sbsec结构 */
    sbsec = kzalloc(sizeof(struct superblock_security_struct), GFP_KERNEL);
    if (!sbsec)
        return -ENOMEM;

    /* 初始化sbsec各个字段 */
    mutex_init(&sbsec->lock);
    INIT_LIST_HEAD(&sbsec->isec_head);
    INIT_HLIST_NODE(&sbsec->sb_list);

    /* 默认的安全上下文来源 */
    sbsec->behavior = SECURITY_FS_USE_NONE;
    sbsec->sid = SECINITSID_FILESYSTEM;  /* 文件系统的初始SID */
    sbsec->def_sid = SECINITSID_FILE;    /* 默认文件SID */

    /* 初始化def_sid映射表 */
    memset(sbsec->mntpoint_sid, 0, sizeof(sbsec->mntpoint_sid));

    /* 设置挂载选项标记 */
    sbsec->flags = 0;
    sbsec->mnt_flags = 0;

    /* 将sbsec关联到sb的security字段 */
    sb->s_security = sbsec;

    return 0;
}
```

selinux_sb_free_security是释放钩子，在超级块释放时回收sbsec：

```c
static void selinux_sb_free_security(struct super_block *sb)
{
    struct superblock_security_struct *sbsec = sb->s_security;

    if (!sbsec)
        return;

    /* 从全局sbsec_list中移除 */
    spin_lock(&sb_security_lock);
    if (!hlist_unhashed(&sbsec->sb_list))
        hlist_del_init(&sbsec->sb_list);
    spin_unlock(&sb_security_lock);

    kfree(sbsec);
    sb->s_security = NULL;
}
```

挂载时selinux_sb_kern_mount完成文件系统类型的安全策略初始化：

```c
static int selinux_sb_kern_mount(struct super_block *sb, int flags, void *data)
{
    int rc;
    struct superblock_security_struct *sbsec = sb->s_security;
    struct dentry *root = sb->s_root;
    struct inode_security_struct *root_isec;

    if (!root)
        return 0;

    root_isec = backing_inode_security(root);

    /* 确定文件系统的安全行为模式 */
    rc = selinux_set_sb_context(sb, flags, data, &sbsec->behavior);
    if (rc)
        return rc;

    /* 根据behavior模式选择SID */
    switch (sbsec->behavior) {
    case SECURITY_FS_USE_XATTR:
        /* 从文件系统xattr中读取安全上下文 */
        break;
    case SECURITY_FS_USE_TASK:
        /* 使用创建者的安全上下文 */
        sbsec->sid = current_sid();
        break;
    case SECURITY_FS_USE_TRANS:
        /* 使用过渡SID */
        sbsec->sid = sbsec->mntpoint_sid[0];
        break;
    case SECURITY_FS_USE_GENFS:
        /* 使用genfs_context策略决定SID */
        sbsec->sid = selinux_genfs_get_sid(sb, root);
        break;
    case SECURITY_FS_USE_NONE:
        /* 不处理xattr，使用默认filesystem SID */
        sbsec->sid = SECINITSID_FILESYSTEM;
        break;
    }

    return 0;
}
```

selinux_set_sb_context根据文件系统特性和mount选项确定sbsec->behavior：

```c
static int selinux_set_sb_context(struct super_block *sb, int flags,
                                  void *data, int *behavior)
{
    struct dentry *root = sb->s_root;
    const struct inode *inode = root->d_inode;

    /* 伪文件系统直接使用def_sid */
    if (sb->s_flags & SB_KERNMOUNT) {
        *behavior = SECURITY_FS_USE_NONE;
        return 0;
    }

    /* 检查文件系统是否支持xattr */
    if (inode->i_opflags & IOP_XATTR) {
        *behavior = SECURITY_FS_USE_XATTR;
    } else if (fs_use_behavior(sb->s_type, behavior)) {
        /* 在策略中查找fs_use_statements定义 */
        return 0;
    } else if (sb->s_magic == TMPFS_MAGIC || sb->s_magic == RAMFS_MAGIC) {
        *behavior = SECURITY_FS_USE_TASK;
    } else {
        *behavior = SECURITY_FS_USE_NONE;
    }

    return 0;
}
```

selinux_sb_clone_mnt_data在mount命名空间复制时克隆sbsec的挂载安全数据：

```c
static int selinux_sb_clone_mnt_data(const struct super_block *oldsb,
                                     struct super_block *newsb)
{
    struct superblock_security_struct *oldsbsec = oldsb->s_security;
    struct superblock_security_struct *newsbsec;

    newsbsec = kmemdup(oldsbsec, sizeof(*oldsbsec), GFP_KERNEL);
    if (!newsbsec)
        return -ENOMEM;

    mutex_init(&newsbsec->lock);
    INIT_LIST_HEAD(&newsbsec->isec_head);
    INIT_HLIST_NODE(&newsbsec->sb_list);

    newsb->s_security = newsbsec;
    return 0;
}
```

selinux_sb_statfs在statfs系统调用时检查是否有权查看文件系统统计信息：

```c
static int selinux_sb_statfs(struct dentry *dentry)
{
    const struct cred *cred = current_cred();
    struct superblock_security_struct *sbsec;
    struct common_audit_data ad;

    sbsec = dentry->d_sb->s_security;

    ad.type = LSM_AUDIT_DATA_DENTRY;
    ad.u.dentry = dentry;

    /* 检查filesystem关联的SID是否允许getattr */
    return avc_has_perm(&selinux_state,
                        cred_sid(cred),
                        sbsec->sid,
                        SECCLASS_FILESYSTEM,
                        FILESYSTEM__GETATTR,
                        &ad);
}
```

SELinux的超级块钩子全集定义在selinux_hooks数组中，包含sb_alloc_security、sb_free_security、sb_kern_mount、sb_clone_mnt_data、sb_copy_mnt_data、sb_statfs和sb_mount等完整的生命周期管理钩子，保证文件系统从挂载到卸载全路径的安全策略执行。
荣障柏园纪咳郝量裙儆蓖链毫毫咐

m.wwi.hzldf.cn/11331.Doc
m.wwi.hzldf.cn/55571.Doc
m.wwu.hzldf.cn/64628.Doc
m.wwu.hzldf.cn/22662.Doc
m.wwu.hzldf.cn/39735.Doc
m.wwu.hzldf.cn/91751.Doc
m.wwu.hzldf.cn/20484.Doc
m.wwu.hzldf.cn/68460.Doc
m.wwu.hzldf.cn/08482.Doc
m.wwu.hzldf.cn/31995.Doc
m.wwu.hzldf.cn/40642.Doc
m.wwu.hzldf.cn/84046.Doc
m.wwu.hzldf.cn/26046.Doc
m.wwu.hzldf.cn/26884.Doc
m.wwu.hzldf.cn/84468.Doc
m.wwu.hzldf.cn/44628.Doc
m.wwu.hzldf.cn/60064.Doc
m.wwu.hzldf.cn/71397.Doc
m.wwu.hzldf.cn/93999.Doc
m.wwu.hzldf.cn/64840.Doc
m.wwu.hzldf.cn/62066.Doc
m.wwu.hzldf.cn/66046.Doc
m.wwy.hzldf.cn/79917.Doc
m.wwy.hzldf.cn/46004.Doc
m.wwy.hzldf.cn/88424.Doc
m.wwy.hzldf.cn/28260.Doc
m.wwy.hzldf.cn/40642.Doc
m.wwy.hzldf.cn/04680.Doc
m.wwy.hzldf.cn/22082.Doc
m.wwy.hzldf.cn/20606.Doc
m.wwy.hzldf.cn/06284.Doc
m.wwy.hzldf.cn/88426.Doc
m.wwy.hzldf.cn/68488.Doc
m.wwy.hzldf.cn/22084.Doc
m.wwy.hzldf.cn/42888.Doc
m.wwy.hzldf.cn/26246.Doc
m.wwy.hzldf.cn/91337.Doc
m.wwy.hzldf.cn/28848.Doc
m.wwy.hzldf.cn/08862.Doc
m.wwy.hzldf.cn/55151.Doc
m.wwy.hzldf.cn/00800.Doc
m.wwy.hzldf.cn/42208.Doc
m.wwt.hzldf.cn/17119.Doc
m.wwt.hzldf.cn/62266.Doc
m.wwt.hzldf.cn/28406.Doc
m.wwt.hzldf.cn/06624.Doc
m.wwt.hzldf.cn/22440.Doc
m.wwt.hzldf.cn/11513.Doc
m.wwt.hzldf.cn/77957.Doc
m.wwt.hzldf.cn/00064.Doc
m.wwt.hzldf.cn/35335.Doc
m.wwt.hzldf.cn/80806.Doc
m.wwt.hzldf.cn/88802.Doc
m.wwt.hzldf.cn/86862.Doc
m.wwt.hzldf.cn/26284.Doc
m.wwt.hzldf.cn/44464.Doc
m.wwt.hzldf.cn/02080.Doc
m.wwt.hzldf.cn/86468.Doc
m.wwt.hzldf.cn/00440.Doc
m.wwt.hzldf.cn/97957.Doc
m.wwt.hzldf.cn/28446.Doc
m.wwt.hzldf.cn/20200.Doc
m.wwr.hzldf.cn/79539.Doc
m.wwr.hzldf.cn/77999.Doc
m.wwr.hzldf.cn/20068.Doc
m.wwr.hzldf.cn/26066.Doc
m.wwr.hzldf.cn/28664.Doc
m.wwr.hzldf.cn/75557.Doc
m.wwr.hzldf.cn/00680.Doc
m.wwr.hzldf.cn/66880.Doc
m.wwr.hzldf.cn/02068.Doc
m.wwr.hzldf.cn/60800.Doc
m.wwr.hzldf.cn/35755.Doc
m.wwr.hzldf.cn/22224.Doc
m.wwr.hzldf.cn/86628.Doc
m.wwr.hzldf.cn/66284.Doc
m.wwr.hzldf.cn/82600.Doc
m.wwr.hzldf.cn/88624.Doc
m.wwr.hzldf.cn/24266.Doc
m.wwr.hzldf.cn/84820.Doc
m.wwr.hzldf.cn/79313.Doc
m.wwr.hzldf.cn/91719.Doc
m.wwe.hzldf.cn/88260.Doc
m.wwe.hzldf.cn/84288.Doc
m.wwe.hzldf.cn/86048.Doc
m.wwe.hzldf.cn/04824.Doc
m.wwe.hzldf.cn/46808.Doc
m.wwe.hzldf.cn/86264.Doc
m.wwe.hzldf.cn/04266.Doc
m.wwe.hzldf.cn/17351.Doc
m.wwe.hzldf.cn/00428.Doc
m.wwe.hzldf.cn/04484.Doc
m.wwe.hzldf.cn/26440.Doc
m.wwe.hzldf.cn/15977.Doc
m.wwe.hzldf.cn/31391.Doc
m.wwe.hzldf.cn/48288.Doc
m.wwe.hzldf.cn/15913.Doc
m.wwe.hzldf.cn/24844.Doc
m.wwe.hzldf.cn/57917.Doc
m.wwe.hzldf.cn/71591.Doc
m.wwe.hzldf.cn/60248.Doc
m.wwe.hzldf.cn/22048.Doc
m.www.hzldf.cn/91179.Doc
m.www.hzldf.cn/73195.Doc
m.www.hzldf.cn/22220.Doc
m.www.hzldf.cn/88404.Doc
m.www.hzldf.cn/64826.Doc
m.www.hzldf.cn/13119.Doc
m.www.hzldf.cn/44244.Doc
m.www.hzldf.cn/68082.Doc
m.www.hzldf.cn/95919.Doc
m.www.hzldf.cn/51119.Doc
m.www.hzldf.cn/46420.Doc
m.www.hzldf.cn/88826.Doc
m.www.hzldf.cn/44042.Doc
m.www.hzldf.cn/02268.Doc
m.www.hzldf.cn/86028.Doc
m.www.hzldf.cn/82688.Doc
m.www.hzldf.cn/22202.Doc
m.www.hzldf.cn/62244.Doc
m.www.hzldf.cn/66882.Doc
m.www.hzldf.cn/60202.Doc
m.wwq.hzldf.cn/22028.Doc
m.wwq.hzldf.cn/82060.Doc
m.wwq.hzldf.cn/00200.Doc
m.wwq.hzldf.cn/46846.Doc
m.wwq.hzldf.cn/59935.Doc
m.wwq.hzldf.cn/46462.Doc
m.wwq.hzldf.cn/02866.Doc
m.wwq.hzldf.cn/88842.Doc
m.wwq.hzldf.cn/44208.Doc
m.wwq.hzldf.cn/00600.Doc
m.wwq.hzldf.cn/00608.Doc
m.wwq.hzldf.cn/31733.Doc
m.wwq.hzldf.cn/39177.Doc
m.wwq.hzldf.cn/48468.Doc
m.wwq.hzldf.cn/26628.Doc
m.wwq.hzldf.cn/66886.Doc
m.wwq.hzldf.cn/71573.Doc
m.wwq.hzldf.cn/42448.Doc
m.wwq.hzldf.cn/62262.Doc
m.wwq.hzldf.cn/26044.Doc
m.wqm.hzldf.cn/37993.Doc
m.wqm.hzldf.cn/35773.Doc
m.wqm.hzldf.cn/84628.Doc
m.wqm.hzldf.cn/26286.Doc
m.wqm.hzldf.cn/04008.Doc
m.wqm.hzldf.cn/28004.Doc
m.wqm.hzldf.cn/42242.Doc
m.wqm.hzldf.cn/48426.Doc
m.wqm.hzldf.cn/37351.Doc
m.wqm.hzldf.cn/88604.Doc
m.wqm.hzldf.cn/75539.Doc
m.wqm.hzldf.cn/75959.Doc
m.wqm.hzldf.cn/64008.Doc
m.wqm.hzldf.cn/48226.Doc
m.wqm.hzldf.cn/24446.Doc
m.wqm.hzldf.cn/33173.Doc
m.wqm.hzldf.cn/88220.Doc
m.wqm.hzldf.cn/08882.Doc
m.wqm.hzldf.cn/95315.Doc
m.wqm.hzldf.cn/99337.Doc
m.wqn.hzldf.cn/68228.Doc
m.wqn.hzldf.cn/55997.Doc
m.wqn.hzldf.cn/00086.Doc
m.wqn.hzldf.cn/40600.Doc
m.wqn.hzldf.cn/55371.Doc
m.wqn.hzldf.cn/82882.Doc
m.wqn.hzldf.cn/82222.Doc
m.wqn.hzldf.cn/64266.Doc
m.wqn.hzldf.cn/00286.Doc
m.wqn.hzldf.cn/20002.Doc
m.wqn.hzldf.cn/40820.Doc
m.wqn.hzldf.cn/20200.Doc
m.wqn.hzldf.cn/20442.Doc
m.wqn.hzldf.cn/77797.Doc
m.wqn.hzldf.cn/99937.Doc
m.wqn.hzldf.cn/82888.Doc
m.wqn.hzldf.cn/04482.Doc
m.wqn.hzldf.cn/46240.Doc
m.wqn.hzldf.cn/04240.Doc
m.wqn.hzldf.cn/00486.Doc
m.wqb.hzldf.cn/37517.Doc
m.wqb.hzldf.cn/75711.Doc
m.wqb.hzldf.cn/17195.Doc
m.wqb.hzldf.cn/86664.Doc
m.wqb.hzldf.cn/64220.Doc
m.wqb.hzldf.cn/08408.Doc
m.wqb.hzldf.cn/60026.Doc
m.wqb.hzldf.cn/17735.Doc
m.wqb.hzldf.cn/79313.Doc
m.wqb.hzldf.cn/66028.Doc
m.wqb.hzldf.cn/11753.Doc
m.wqb.hzldf.cn/80202.Doc
m.wqb.hzldf.cn/80066.Doc
m.wqb.hzldf.cn/73515.Doc
m.wqb.hzldf.cn/77595.Doc
m.wqb.hzldf.cn/73575.Doc
m.wqb.hzldf.cn/64480.Doc
m.wqb.hzldf.cn/40860.Doc
m.wqb.hzldf.cn/44226.Doc
m.wqb.hzldf.cn/77171.Doc
m.wqv.hzldf.cn/40088.Doc
m.wqv.hzldf.cn/62608.Doc
m.wqv.hzldf.cn/99397.Doc
m.wqv.hzldf.cn/04604.Doc
m.wqv.hzldf.cn/48822.Doc
m.wqv.hzldf.cn/88042.Doc
m.wqv.hzldf.cn/48644.Doc
m.wqv.hzldf.cn/57319.Doc
m.wqv.hzldf.cn/91919.Doc
m.wqv.hzldf.cn/17575.Doc
m.wqv.hzldf.cn/80248.Doc
m.wqv.hzldf.cn/46666.Doc
m.wqv.hzldf.cn/37313.Doc
m.wqv.hzldf.cn/42624.Doc
m.wqv.hzldf.cn/71313.Doc
m.wqv.hzldf.cn/68264.Doc
m.wqv.hzldf.cn/88020.Doc
m.wqv.hzldf.cn/62686.Doc
m.wqv.hzldf.cn/44202.Doc
m.wqv.hzldf.cn/06626.Doc
m.wqc.hzldf.cn/53559.Doc
m.wqc.hzldf.cn/82260.Doc
m.wqc.hzldf.cn/57395.Doc
m.wqc.hzldf.cn/99911.Doc
m.wqc.hzldf.cn/48420.Doc
m.wqc.hzldf.cn/20802.Doc
m.wqc.hzldf.cn/13959.Doc
m.wqc.hzldf.cn/20004.Doc
m.wqc.hzldf.cn/71959.Doc
m.wqc.hzldf.cn/28460.Doc
m.wqc.hzldf.cn/31371.Doc
m.wqc.hzldf.cn/97997.Doc
m.wqc.hzldf.cn/44224.Doc
m.wqc.hzldf.cn/22268.Doc
m.wqc.hzldf.cn/33731.Doc
m.wqc.hzldf.cn/28642.Doc
m.wqc.hzldf.cn/40806.Doc
m.wqc.hzldf.cn/20862.Doc
m.wqc.hzldf.cn/04488.Doc
m.wqc.hzldf.cn/42008.Doc
m.wqx.hzldf.cn/79917.Doc
m.wqx.hzldf.cn/15157.Doc
m.wqx.hzldf.cn/68280.Doc
m.wqx.hzldf.cn/04204.Doc
m.wqx.hzldf.cn/37515.Doc
m.wqx.hzldf.cn/13351.Doc
m.wqx.hzldf.cn/46204.Doc
m.wqx.hzldf.cn/26068.Doc
m.wqx.hzldf.cn/08060.Doc
m.wqx.hzldf.cn/08444.Doc
m.wqx.hzldf.cn/00846.Doc
m.wqx.hzldf.cn/11153.Doc
m.wqx.hzldf.cn/80688.Doc
m.wqx.hzldf.cn/17339.Doc
m.wqx.hzldf.cn/73315.Doc
m.wqx.hzldf.cn/42240.Doc
m.wqx.hzldf.cn/62448.Doc
m.wqx.hzldf.cn/04664.Doc
m.wqx.hzldf.cn/71519.Doc
m.wqx.hzldf.cn/82488.Doc
m.wqz.hzldf.cn/28248.Doc
m.wqz.hzldf.cn/20440.Doc
m.wqz.hzldf.cn/82266.Doc
m.wqz.hzldf.cn/91939.Doc
m.wqz.hzldf.cn/55995.Doc
m.wqz.hzldf.cn/02684.Doc
m.wqz.hzldf.cn/26260.Doc
m.wqz.hzldf.cn/66888.Doc
m.wqz.hzldf.cn/08488.Doc
m.wqz.hzldf.cn/93573.Doc
m.wqz.hzldf.cn/62088.Doc
m.wqz.hzldf.cn/62062.Doc
m.wqz.hzldf.cn/33131.Doc
m.wqz.hzldf.cn/26222.Doc
m.wqz.hzldf.cn/31157.Doc
m.wqz.hzldf.cn/88048.Doc
m.wqz.hzldf.cn/53117.Doc
m.wqz.hzldf.cn/80606.Doc
m.wqz.hzldf.cn/60020.Doc
m.wqz.hzldf.cn/15591.Doc
m.wql.hzldf.cn/84862.Doc
m.wql.hzldf.cn/60840.Doc
m.wql.hzldf.cn/44620.Doc
m.wql.hzldf.cn/39559.Doc
m.wql.hzldf.cn/40844.Doc
m.wql.hzldf.cn/51977.Doc
m.wql.hzldf.cn/55551.Doc
m.wql.hzldf.cn/44440.Doc
m.wql.hzldf.cn/93197.Doc
m.wql.hzldf.cn/04024.Doc
m.wql.hzldf.cn/22468.Doc
m.wql.hzldf.cn/20224.Doc
m.wql.hzldf.cn/75917.Doc
m.wql.hzldf.cn/82642.Doc
m.wql.hzldf.cn/88086.Doc
m.wql.hzldf.cn/86866.Doc
m.wql.hzldf.cn/11133.Doc
m.wql.hzldf.cn/33791.Doc
m.wql.hzldf.cn/73197.Doc
m.wql.hzldf.cn/08244.Doc
m.wqk.hzldf.cn/48262.Doc
m.wqk.hzldf.cn/77155.Doc
m.wqk.hzldf.cn/64264.Doc
m.wqk.hzldf.cn/19599.Doc
m.wqk.hzldf.cn/17715.Doc
m.wqk.hzldf.cn/11595.Doc
m.wqk.hzldf.cn/02660.Doc
m.wqk.hzldf.cn/48606.Doc
m.wqk.hzldf.cn/06000.Doc
m.wqk.hzldf.cn/48686.Doc
m.wqk.hzldf.cn/64600.Doc
m.wqk.hzldf.cn/31195.Doc
m.wqk.hzldf.cn/80286.Doc
m.wqk.hzldf.cn/46826.Doc
m.wqk.hzldf.cn/42464.Doc
m.wqk.hzldf.cn/68446.Doc
m.wqk.hzldf.cn/46480.Doc
m.wqk.hzldf.cn/42422.Doc
m.wqk.hzldf.cn/08280.Doc
m.wqk.hzldf.cn/42626.Doc
m.wqj.hzldf.cn/82608.Doc
m.wqj.hzldf.cn/20488.Doc
m.wqj.hzldf.cn/44468.Doc
m.wqj.hzldf.cn/95737.Doc
m.wqj.hzldf.cn/88446.Doc
m.wqj.hzldf.cn/68086.Doc
m.wqj.hzldf.cn/04626.Doc
m.wqj.hzldf.cn/68682.Doc
m.wqj.hzldf.cn/82226.Doc
m.wqj.hzldf.cn/06846.Doc
m.wqj.hzldf.cn/37311.Doc
m.wqj.hzldf.cn/62864.Doc
m.wqj.hzldf.cn/93357.Doc
m.wqj.hzldf.cn/06804.Doc
m.wqj.hzldf.cn/64444.Doc
m.wqj.hzldf.cn/93911.Doc
m.wqj.hzldf.cn/15197.Doc
m.wqj.hzldf.cn/06868.Doc
m.wqj.hzldf.cn/71791.Doc
m.wqj.hzldf.cn/44282.Doc
m.wqh.hzldf.cn/08224.Doc
m.wqh.hzldf.cn/08482.Doc
m.wqh.hzldf.cn/53399.Doc
m.wqh.hzldf.cn/35195.Doc
m.wqh.hzldf.cn/04408.Doc
m.wqh.hzldf.cn/80288.Doc
m.wqh.hzldf.cn/42448.Doc
m.wqh.hzldf.cn/37915.Doc
m.wqh.hzldf.cn/40606.Doc
m.wqh.hzldf.cn/20628.Doc
m.wqh.hzldf.cn/84864.Doc
m.wqh.hzldf.cn/82088.Doc
m.wqh.hzldf.cn/93553.Doc
m.wqh.hzldf.cn/26824.Doc
m.wqh.hzldf.cn/00800.Doc
m.wqh.hzldf.cn/02244.Doc
m.wqh.hzldf.cn/73597.Doc
m.wqh.hzldf.cn/00620.Doc
m.wqh.hzldf.cn/28666.Doc
m.wqh.hzldf.cn/59735.Doc
m.wqg.hzldf.cn/88644.Doc
m.wqg.hzldf.cn/17975.Doc
m.wqg.hzldf.cn/20604.Doc
m.wqg.hzldf.cn/44622.Doc
m.wqg.hzldf.cn/51319.Doc
m.wqg.hzldf.cn/24060.Doc
m.wqg.hzldf.cn/40664.Doc
m.wqg.hzldf.cn/02824.Doc
m.wqg.hzldf.cn/33113.Doc
m.wqg.hzldf.cn/22826.Doc
m.wqg.hzldf.cn/62846.Doc
m.wqg.hzldf.cn/19953.Doc
m.wqg.hzldf.cn/02686.Doc
m.wqg.hzldf.cn/02842.Doc
m.wqg.hzldf.cn/59151.Doc
m.wqg.hzldf.cn/82860.Doc
m.wqg.hzldf.cn/04000.Doc
m.wqg.hzldf.cn/40866.Doc
m.wqg.hzldf.cn/88668.Doc
m.wqg.hzldf.cn/97933.Doc
m.wqf.hzldf.cn/35939.Doc
m.wqf.hzldf.cn/60866.Doc
m.wqf.hzldf.cn/37573.Doc
m.wqf.hzldf.cn/08002.Doc
m.wqf.hzldf.cn/22022.Doc
m.wqf.hzldf.cn/04006.Doc
m.wqf.hzldf.cn/24682.Doc
m.wqf.hzldf.cn/22688.Doc
m.wqf.hzldf.cn/46668.Doc
m.wqf.hzldf.cn/28666.Doc
m.wqf.hzldf.cn/24444.Doc
m.wqf.hzldf.cn/66080.Doc
m.wqf.hzldf.cn/46822.Doc
