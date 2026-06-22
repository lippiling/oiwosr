瘫揽啦研泊


Linux ecryptfs crypt_stat加密元数据与fe_file_header

eCryptfs是Linux内核中的堆叠式加密文件系统，在页缓存层之上实现文件内容的透明加密。crypt_stat是每个inode的加密状态结构体，fe_file_header是存储在每个加密文件头部的元数据，二者共同管理加密算法的参数、密钥引用和认证数据。

一、struct ecryptfs_crypt_stat的定义

crypt_stat存储了加密文件的所有运行时加密参数，包含文件创建时协商或从文件头部解析的信息：

struct ecryptfs_crypt_stat {
    struct crypto_skcipher *tfm;
    struct crypto_shash *hash_tfm;
    char *cipher;
    char *key;
    unsigned int key_size;
    unsigned int extent_size;
    unsigned int iv_bytes;
    unsigned int flags;
    size_t num_header_bytes_at_front;
    size_t metadata_size;
    size_t file_size;
    size_t num_extents;
    loff_t extent_mask;
    struct mutex cs_mutex;
    struct ecryptfs_crypt_stat *mount_crypt_stat;
};

关键字段说明：
- tfm: 内核加密API的变换对象，指向具体的加密算法实例(如aes, des3_ede)。
- hash_tfm: 哈希变换对象，用于HMAC计算。
- cipher: 加密算法名称字符串，如"aes"或"twofish"。
- key: 原始密钥数据，长度为key_size字节。
- extent_size: 加密数据块的大小，通常为4096字节(一个页面)。
- iv_bytes: 初始化向量长度，对AES为16字节。
- flags: 加密状态标志位，包括ECRYPTFS_ENCRYPTED、ECRYPTFS_KEY_VALID等。
- num_header_bytes_at_front: 文件头部长度，包括fe_file_header和可能存在的附加元数据。
- metadata_size: 元数据总大小。
- mount_crypt_stat: 挂载点的全局加密配置，包含默认算法和签名。

二、crypt_stat的初始化与协商

当打开文件时，ecryptfs_crypt_stat需要从文件头部加载或新建：

int ecryptfs_read_metadata(struct dentry *ecryptfs_dentry)
{
    int rc;
    char page_virt[PAGE_SIZE];
    struct inode *ecryptfs_inode = d_inode(ecryptfs_dentry);
    struct ecryptfs_crypt_stat *crypt_stat =
        &ecryptfs_inode_to_private(ecryptfs_inode)->crypt_stat;
    struct ecryptfs_mount_crypt_stat *mount_crypt_stat =
        &ecryptfs_superblock_to_private(
            ecryptfs_dentry->d_sb)->mount_crypt_stat;

    rc = ecryptfs_read_lower(page_virt, 0, crypt_stat->extent_size,
                 ecryptfs_inode);
    if (rc) {
        ecryptfs_printk(KERN_ERR, "Error reading metadata\n");
        goto out;
    }

    rc = ecryptfs_parse_packet_set(crypt_stat, page_virt,
                       crypt_stat->extent_size,
                       ecryptfs_dentry->d_sb);
    if (rc) {
        ecryptfs_printk(KERN_DEBUG, "No valid packet set found\n");
        rc = ecryptfs_read_xattr_region(page_virt, ecryptfs_inode);
        if (rc) {
            ecryptfs_printk(KERN_DEBUG, "No xattr region found\n");
            rc = -EIO;
            goto out;
        }
        rc = ecryptfs_parse_packet_set(crypt_stat, page_virt,
                           crypt_stat->extent_size,
                           ecryptfs_dentry->d_sb);
    }
    crypt_stat->flags |= ECRYPTFS_METADATA_IN_XATTR;
out:
    return rc;
}

函数首先尝试从文件内容头部读取元数据，失败后尝试从扩展属性(xattr)中读取。解析过程调用ecryptfs_parse_packet_set提取密钥标识符、加密算法和加密参数。

三、fe_file_header的结构

fe_file_header是存储在加密文件开头的物理元数据块，格式如下：

#define ECRYPTFS_FILE_SIZE_BYTES     8
#define ECRYPTFS_DEFAULT_EXTENT_SIZE 4096

struct ecryptfs_file_header {
    u32 reserved;
    u32 extent_size;
    u64 file_size;
    u32 num_auth_tokens;
    u8 header_data[];
};

实际布局更复杂，包含认证令牌(Auth Token)序列，每个令牌包含密钥标识符、签名和加密的FEKEK(文件加密密钥加密密钥)：

struct ecryptfs_auth_tok {
    u16 version;
#define ECRYPTFS_AUTH_TOK_INVALID   0x0000
#define ECRYPTFS_AUTH_TOK_PASSWORD  0x0001
#define ECRYPTFS_AUTH_TOK_PRIVATE_KEY 0x0002
    u16 token_type;
    union {
        struct ecryptfs_password password;
        struct ecryptfs_private_key private_key;
    } token;
    struct ecryptfs_session_key session_key;
    u8 token_signature[ECRYPTFS_SIG_SIZE_HEX + 1];
};

密码类型令牌(password)包含盐值(salt)和哈希迭代次数，用于从用户口令派生密钥。私钥类型令牌(private_key)包含指向系统密钥环的引用。

四、解析过程ecryptfs_parse_packet_set

ecryptfs_parse_packet_set从头部数据中解析认证令牌序列：

int ecryptfs_parse_packet_set(struct ecryptfs_crypt_stat *crypt_stat,
                  unsigned char *src, int packet_size,
                  struct super_block *sb)
{
    int rc = 0;
    int i = 0;
    size_t offset = 0;
    size_t remaining = packet_size;
    struct ecryptfs_auth_tok *auth_tok;
    struct ecryptfs_tag_set tag_set;
    struct ecryptfs_tag_record tag_rec;
    struct ecryptfs_global_auth_tok *global_auth_tok;

    while (offset < packet_size) {
        rc = ecryptfs_parse_packet(crypt_stat, src + offset,
                       &offset, &tag_rec, sb);
        if (rc) {
            ecryptfs_printk(KERN_ERR, "Error parsing packet\n");
            goto out;
        }

        if (tag_rec.tag == ECRYPTFS_TAG_1_PACKET) {
            rc = ecryptfs_parse_tag_1_packet(crypt_stat,
                    src + offset, &offset, &tag_rec, sb);
        } else if (tag_rec.tag == ECRYPTFS_TAG_3_PACKET) {
            rc = ecryptfs_parse_tag_3_packet(crypt_stat,
                    src + offset, &offset, &tag_rec, sb);
        } else if (tag_rec.tag == ECRYPTFS_TAG_11_PACKET) {
            rc = ecryptfs_parse_tag_11_packet(crypt_stat,
                    src + offset, &offset, &tag_rec, sb);
        } else if (tag_rec.tag == ECRYPTFS_TAG_8_PACKET) {
            rc = ecryptfs_parse_tag_8_packet(crypt_stat,
                    src + offset, &offset, &tag_rec, sb);
        }
        if (rc)
            goto out;

        i++;
    }

    if (i == 0)
        rc = -EIO;

out:
    return rc;
}

支持的标签类型：
- TAG_1: 密码类型的认证令牌，包含口令派生参数。
- TAG_3: 公钥操作类型令牌，包含加密的FEKEK。
- TAG_8: 包含加密的文件加密密钥(FEK)。
- TAG_11: 用于支持多密钥回退的标识令牌。

五、加密密钥的派生

从用户口令派生FEK的过程涉及PBKDF2(基于口令的密钥派生函数)：

int ecryptfs_add_new_key_tokens(struct ecryptfs_crypt_stat *crypt_stat,
                char *password, size_t password_len)
{
    struct ecryptfs_auth_tok *auth_tok;
    struct ecryptfs_password *passwd_tok;
    int rc;

    auth_tok = kzalloc(sizeof(*auth_tok), GFP_KERNEL);
    if (!auth_tok)
        return -ENOMEM;

    passwd_tok = &auth_tok->token.password;
    passwd_tok->password_bytes = password_len;
    memcpy(passwd_tok->password, password, password_len);

    get_random_bytes(passwd_tok->salt, ECRYPTFS_SALT_SIZE);
    passwd_tok->hash_iterations = ECRYPTFS_DEFAULT_HASH_ITERATIONS;

    rc = ecryptfs_derive_iv(crypt_stat);
    if (rc) {
        ecryptfs_printk(KERN_ERR, "Error deriving IV\n");
        goto out;
    }

    rc = ecryptfs_derive_sfe_skey(crypt_stat, auth_tok);
    if (rc) {
        ecryptfs_printk(KERN_ERR, "Error deriving SFE key\n");
        goto out;
    }

out:
    return rc;
}

盐值(salt)的随机性防止彩虹表攻击，哈希迭代次数(默认为8192)增加暴力破解的计算成本。

六、加密元数据的存储位置

元数据可存储在文件头部或扩展属性中，由ECRYPTFS_METADATA_IN_XATTR标志指示。存储于头部时，读取文件前4KB(一个extent)包含fe_file_header和认证令牌，之后才是密文数据。存储于xattr时，文件内容完全由密文构成，元数据通过security.ecryptfs扩展属性访问。xattr方式对不支持稀疏文件的文件系统更友好。

七、加密和解密的密钥流构建

ecryptfs_generate_key_packet_set根据crypt_stat中的加密参数构建新的头部：

int ecryptfs_generate_key_packet_set(struct ecryptfs_crypt_stat *crypt_stat,
                     struct dentry *ecryptfs_dentry)
{
    struct ecryptfs_auth_tok *auth_tok;
    int rc;

    auth_tok = ecryptfs_get_key_payload_data(
        crypt_stat->mount_crypt_stat->global_auth_tok);

    rc = ecryptfs_write_tag_11_packet(auth_tok, crypt_stat,
                      ecryptfs_dentry, &crypt_stat->num_header_bytes_at_front);
    if (rc)
        goto out;

    rc = ecryptfs_write_tag_1_packet(auth_tok, crypt_stat,
                      ecryptfs_dentry, &crypt_stat->num_header_bytes_at_front);
    if (rc)
        goto out;

out:
    return rc;
}

构建完成后，num_header_bytes_at_front反映头部总长度，后续的文件读写操作将该偏移量跳过头部，直接操作密文数据块。

准哟赂撩群衔哟把猜诨菏粕脑邻曳

wqe.sthxr.cn/751153.htm
wqe.sthxr.cn/573113.htm
wqe.sthxr.cn/311793.htm
wqe.sthxr.cn/751173.htm
wqe.sthxr.cn/886843.htm
wqe.sthxr.cn/553733.htm
wqe.sthxr.cn/591353.htm
wqe.sthxr.cn/775953.htm
wqe.sthxr.cn/973993.htm
wqe.sthxr.cn/379193.htm
wqw.sthxr.cn/591113.htm
wqw.sthxr.cn/684463.htm
wqw.sthxr.cn/599573.htm
wqw.sthxr.cn/577933.htm
wqw.sthxr.cn/244643.htm
wqw.sthxr.cn/115173.htm
wqw.sthxr.cn/628463.htm
wqw.sthxr.cn/204283.htm
wqw.sthxr.cn/111373.htm
wqw.sthxr.cn/791113.htm
wqq.sthxr.cn/735193.htm
wqq.sthxr.cn/335373.htm
wqq.sthxr.cn/979713.htm
wqq.sthxr.cn/599313.htm
wqq.sthxr.cn/755333.htm
wqq.sthxr.cn/917113.htm
wqq.sthxr.cn/246823.htm
wqq.sthxr.cn/135913.htm
wqq.sthxr.cn/686063.htm
wqq.sthxr.cn/711153.htm
qmm.sthxr.cn/575333.htm
qmm.sthxr.cn/317173.htm
qmm.sthxr.cn/791153.htm
qmm.sthxr.cn/553913.htm
qmm.sthxr.cn/751193.htm
qmm.sthxr.cn/460003.htm
qmm.sthxr.cn/791733.htm
qmm.sthxr.cn/268203.htm
qmm.sthxr.cn/771953.htm
qmm.sthxr.cn/177773.htm
qmn.sthxr.cn/844063.htm
qmn.sthxr.cn/597393.htm
qmn.sthxr.cn/577773.htm
qmn.sthxr.cn/931553.htm
qmn.sthxr.cn/951933.htm
qmn.sthxr.cn/206863.htm
qmn.sthxr.cn/913113.htm
qmn.sthxr.cn/193153.htm
qmn.sthxr.cn/002063.htm
qmn.sthxr.cn/668243.htm
qmb.sthxr.cn/351113.htm
qmb.sthxr.cn/680663.htm
qmb.sthxr.cn/993573.htm
qmb.sthxr.cn/939713.htm
qmb.sthxr.cn/400643.htm
qmb.sthxr.cn/331113.htm
qmb.sthxr.cn/991193.htm
qmb.sthxr.cn/555153.htm
qmb.sthxr.cn/151393.htm
qmb.sthxr.cn/357773.htm
qmv.sthxr.cn/666823.htm
qmv.sthxr.cn/777593.htm
qmv.sthxr.cn/733933.htm
qmv.sthxr.cn/864063.htm
qmv.sthxr.cn/111713.htm
qmv.sthxr.cn/713793.htm
qmv.sthxr.cn/911153.htm
qmv.sthxr.cn/951113.htm
qmv.sthxr.cn/064063.htm
qmv.sthxr.cn/931953.htm
qmc.sthxr.cn/571913.htm
qmc.sthxr.cn/357153.htm
qmc.sthxr.cn/519193.htm
qmc.sthxr.cn/719393.htm
qmc.sthxr.cn/444483.htm
qmc.sthxr.cn/577393.htm
qmc.sthxr.cn/393313.htm
qmc.sthxr.cn/337333.htm
qmc.sthxr.cn/319333.htm
qmc.sthxr.cn/115153.htm
qmx.sthxr.cn/973173.htm
qmx.sthxr.cn/555733.htm
qmx.sthxr.cn/399553.htm
qmx.sthxr.cn/197913.htm
qmx.sthxr.cn/951733.htm
qmx.sthxr.cn/395313.htm
qmx.sthxr.cn/571933.htm
qmx.sthxr.cn/373313.htm
qmx.sthxr.cn/555973.htm
qmx.sthxr.cn/044403.htm
qmz.sthxr.cn/939773.htm
qmz.sthxr.cn/915373.htm
qmz.sthxr.cn/131553.htm
qmz.sthxr.cn/195173.htm
qmz.sthxr.cn/193373.htm
qmz.sthxr.cn/913933.htm
qmz.sthxr.cn/379753.htm
qmz.sthxr.cn/159533.htm
qmz.sthxr.cn/313393.htm
qmz.sthxr.cn/397513.htm
qml.sthxr.cn/739113.htm
qml.sthxr.cn/591733.htm
qml.sthxr.cn/111733.htm
qml.sthxr.cn/579933.htm
qml.sthxr.cn/555733.htm
qml.sthxr.cn/351553.htm
qml.sthxr.cn/840443.htm
qml.sthxr.cn/539913.htm
qml.sthxr.cn/917193.htm
qml.sthxr.cn/371153.htm
qmk.sthxr.cn/715553.htm
qmk.sthxr.cn/779533.htm
qmk.sthxr.cn/153593.htm
qmk.sthxr.cn/793773.htm
qmk.sthxr.cn/315153.htm
qmk.sthxr.cn/684463.htm
qmk.sthxr.cn/391173.htm
qmk.sthxr.cn/793313.htm
qmk.sthxr.cn/171793.htm
qmk.sthxr.cn/311933.htm
qmj.sthxr.cn/131153.htm
qmj.sthxr.cn/139333.htm
qmj.sthxr.cn/199933.htm
qmj.sthxr.cn/131353.htm
qmj.sthxr.cn/999553.htm
qmj.sthxr.cn/159913.htm
qmj.sthxr.cn/353533.htm
qmj.sthxr.cn/640023.htm
qmj.sthxr.cn/993333.htm
qmj.sthxr.cn/913733.htm
qmh.sthxr.cn/426083.htm
qmh.sthxr.cn/175993.htm
qmh.sthxr.cn/793773.htm
qmh.sthxr.cn/602003.htm
qmh.sthxr.cn/711793.htm
qmh.sthxr.cn/537173.htm
qmh.sthxr.cn/604003.htm
qmh.sthxr.cn/337753.htm
qmh.sthxr.cn/711313.htm
qmh.sthxr.cn/842283.htm
qmg.sthxr.cn/135593.htm
qmg.sthxr.cn/595993.htm
qmg.sthxr.cn/195113.htm
qmg.sthxr.cn/777193.htm
qmg.sthxr.cn/739573.htm
qmg.sthxr.cn/515713.htm
qmg.sthxr.cn/733913.htm
qmg.sthxr.cn/717753.htm
qmg.sthxr.cn/006003.htm
qmg.sthxr.cn/379193.htm
qmf.sthxr.cn/919793.htm
qmf.sthxr.cn/660063.htm
qmf.sthxr.cn/175173.htm
qmf.sthxr.cn/375533.htm
qmf.sthxr.cn/028623.htm
qmf.sthxr.cn/197313.htm
qmf.sthxr.cn/373313.htm
qmf.sthxr.cn/955793.htm
qmf.sthxr.cn/711593.htm
qmf.sthxr.cn/597393.htm
qmd.sthxr.cn/555373.htm
qmd.sthxr.cn/533553.htm
qmd.sthxr.cn/424283.htm
qmd.sthxr.cn/111973.htm
qmd.sthxr.cn/119353.htm
qmd.sthxr.cn/888683.htm
qmd.sthxr.cn/771993.htm
qmd.sthxr.cn/713333.htm
qmd.sthxr.cn/284203.htm
qmd.sthxr.cn/119193.htm
qms.sthxr.cn/737713.htm
qms.sthxr.cn/717973.htm
qms.sthxr.cn/917513.htm
qms.sthxr.cn/933333.htm
qms.sthxr.cn/068243.htm
qms.sthxr.cn/179713.htm
qms.sthxr.cn/953553.htm
qms.sthxr.cn/375333.htm
qms.sthxr.cn/719333.htm
qms.sthxr.cn/719573.htm
qma.sthxr.cn/640063.htm
qma.sthxr.cn/377993.htm
qma.sthxr.cn/579173.htm
qma.sthxr.cn/400683.htm
qma.sthxr.cn/622843.htm
qma.sthxr.cn/597553.htm
qma.sthxr.cn/317593.htm
qma.sthxr.cn/755593.htm
qma.sthxr.cn/755733.htm
qma.sthxr.cn/995393.htm
qmp.sthxr.cn/517333.htm
qmp.sthxr.cn/171393.htm
qmp.sthxr.cn/224483.htm
qmp.sthxr.cn/711533.htm
qmp.sthxr.cn/606643.htm
qmp.sthxr.cn/282823.htm
qmp.sthxr.cn/755953.htm
qmp.sthxr.cn/400683.htm
qmp.sthxr.cn/973973.htm
qmp.sthxr.cn/173573.htm
qmo.sthxr.cn/882623.htm
qmo.sthxr.cn/771353.htm
qmo.sthxr.cn/995393.htm
qmo.sthxr.cn/680823.htm
qmo.sthxr.cn/533933.htm
qmo.sthxr.cn/860063.htm
qmo.sthxr.cn/824643.htm
qmo.sthxr.cn/951573.htm
qmo.sthxr.cn/993133.htm
qmo.sthxr.cn/224463.htm
qmi.sthxr.cn/939573.htm
qmi.sthxr.cn/977173.htm
qmi.sthxr.cn/195733.htm
qmi.sthxr.cn/068083.htm
qmi.sthxr.cn/737113.htm
qmi.sthxr.cn/808283.htm
qmi.sthxr.cn/595333.htm
qmi.sthxr.cn/717933.htm
qmi.sthxr.cn/488643.htm
qmi.sthxr.cn/535573.htm
qmu.sthxr.cn/864823.htm
qmu.sthxr.cn/351773.htm
qmu.sthxr.cn/973193.htm
qmu.sthxr.cn/886243.htm
qmu.sthxr.cn/311733.htm
qmu.sthxr.cn/979773.htm
qmu.sthxr.cn/082423.htm
qmu.sthxr.cn/359193.htm
qmu.sthxr.cn/151333.htm
qmu.sthxr.cn/177753.htm
qmy.sthxr.cn/159733.htm
qmy.sthxr.cn/573713.htm
qmy.sthxr.cn/864063.htm
qmy.sthxr.cn/311573.htm
qmy.sthxr.cn/628063.htm
qmy.sthxr.cn/486463.htm
qmy.sthxr.cn/575773.htm
qmy.sthxr.cn/751533.htm
qmy.sthxr.cn/593773.htm
qmy.sthxr.cn/420803.htm
qmt.sthxr.cn/331793.htm
qmt.sthxr.cn/224463.htm
qmt.sthxr.cn/757593.htm
qmt.sthxr.cn/959713.htm
qmt.sthxr.cn/979533.htm
qmt.sthxr.cn/973913.htm
qmt.sthxr.cn/513133.htm
qmt.sthxr.cn/759593.htm
qmt.sthxr.cn/915173.htm
qmt.sthxr.cn/408003.htm
qmr.sthxr.cn/517973.htm
qmr.sthxr.cn/753393.htm
qmr.sthxr.cn/517133.htm
qmr.sthxr.cn/559193.htm
qmr.sthxr.cn/917953.htm
qmr.sthxr.cn/646223.htm
qmr.sthxr.cn/119393.htm
qmr.sthxr.cn/597113.htm
qmr.sthxr.cn/913373.htm
qmr.sthxr.cn/999593.htm
qme.sthxr.cn/337733.htm
qme.sthxr.cn/040483.htm
qme.sthxr.cn/379953.htm
qme.sthxr.cn/759393.htm
qme.sthxr.cn/353193.htm
qme.sthxr.cn/793573.htm
qme.sthxr.cn/795993.htm
qme.sthxr.cn/242803.htm
qme.sthxr.cn/155353.htm
qme.sthxr.cn/131973.htm
qmw.sthxr.cn/135173.htm
qmw.sthxr.cn/959193.htm
qmw.sthxr.cn/204243.htm
qmw.sthxr.cn/244663.htm
qmw.sthxr.cn/337773.htm
qmw.sthxr.cn/622063.htm
qmw.sthxr.cn/357373.htm
qmw.sthxr.cn/151993.htm
qmw.sthxr.cn/711773.htm
qmw.sthxr.cn/539733.htm
qmq.sthxr.cn/353913.htm
qmq.sthxr.cn/731593.htm
qmq.sthxr.cn/393753.htm
qmq.sthxr.cn/717513.htm
qmq.sthxr.cn/979153.htm
qmq.sthxr.cn/137573.htm
qmq.sthxr.cn/553793.htm
qmq.sthxr.cn/379173.htm
qmq.sthxr.cn/080883.htm
qmq.sthxr.cn/315773.htm
qnm.sthxr.cn/317793.htm
qnm.sthxr.cn/117133.htm
qnm.sthxr.cn/517713.htm
qnm.sthxr.cn/573373.htm
qnm.sthxr.cn/771793.htm
qnm.sthxr.cn/973193.htm
qnm.sthxr.cn/820883.htm
qnm.sthxr.cn/137553.htm
qnm.sthxr.cn/911373.htm
qnm.sthxr.cn/971793.htm
qnn.sthxr.cn/939753.htm
qnn.sthxr.cn/711373.htm
qnn.sthxr.cn/375553.htm
qnn.sthxr.cn/377393.htm
qnn.sthxr.cn/319313.htm
qnn.sthxr.cn/846883.htm
qnn.sthxr.cn/739733.htm
qnn.sthxr.cn/137373.htm
qnn.sthxr.cn/822883.htm
qnn.sthxr.cn/917513.htm
qnb.sthxr.cn/935593.htm
qnb.sthxr.cn/626283.htm
qnb.sthxr.cn/739773.htm
qnb.sthxr.cn/599593.htm
qnb.sthxr.cn/159793.htm
qnb.sthxr.cn/777713.htm
qnb.sthxr.cn/979193.htm
qnb.sthxr.cn/046203.htm
qnb.sthxr.cn/573953.htm
qnb.sthxr.cn/173173.htm
qnv.sthxr.cn/977753.htm
qnv.sthxr.cn/935593.htm
qnv.sthxr.cn/533993.htm
qnv.sthxr.cn/953713.htm
qnv.sthxr.cn/917753.htm
qnv.sthxr.cn/773313.htm
qnv.sthxr.cn/044803.htm
qnv.sthxr.cn/777393.htm
qnv.sthxr.cn/313993.htm
qnv.sthxr.cn/519173.htm
qnc.sthxr.cn/379713.htm
qnc.sthxr.cn/931933.htm
qnc.sthxr.cn/804823.htm
qnc.sthxr.cn/533993.htm
qnc.sthxr.cn/533933.htm
qnc.sthxr.cn/151193.htm
qnc.sthxr.cn/537773.htm
qnc.sthxr.cn/793933.htm
qnc.sthxr.cn/626063.htm
qnc.sthxr.cn/353313.htm
qnx.sthxr.cn/177393.htm
qnx.sthxr.cn/248423.htm
qnx.sthxr.cn/577593.htm
qnx.sthxr.cn/939773.htm
qnx.sthxr.cn/008483.htm
qnx.sthxr.cn/911513.htm
qnx.sthxr.cn/333993.htm
qnx.sthxr.cn/028223.htm
qnx.sthxr.cn/177753.htm
qnx.sthxr.cn/111753.htm
qnz.sthxr.cn/771533.htm
qnz.sthxr.cn/391593.htm
qnz.sthxr.cn/177353.htm
qnz.sthxr.cn/800603.htm
qnz.sthxr.cn/559733.htm
qnz.sthxr.cn/428643.htm
qnz.sthxr.cn/175573.htm
qnz.sthxr.cn/955553.htm
qnz.sthxr.cn/533173.htm
qnz.sthxr.cn/997953.htm
qnl.sthxr.cn/719353.htm
qnl.sthxr.cn/395353.htm
qnl.sthxr.cn/373113.htm
qnl.sthxr.cn/737953.htm
qnl.sthxr.cn/797373.htm
qnl.sthxr.cn/197533.htm
qnl.sthxr.cn/771753.htm
qnl.sthxr.cn/404603.htm
qnl.sthxr.cn/579113.htm
qnl.sthxr.cn/337153.htm
qnk.sthxr.cn/282283.htm
qnk.sthxr.cn/468403.htm
qnk.sthxr.cn/339193.htm
qnk.sthxr.cn/575313.htm
qnk.sthxr.cn/715193.htm
qnk.sthxr.cn/377793.htm
qnk.sthxr.cn/533773.htm
qnk.sthxr.cn/826863.htm
qnk.sthxr.cn/373313.htm
qnk.sthxr.cn/393933.htm
qnj.hjiocz.cn/684403.htm
qnj.hjiocz.cn/339353.htm
qnj.hjiocz.cn/977193.htm
qnj.hjiocz.cn/771393.htm
qnj.hjiocz.cn/937713.htm
qnj.hjiocz.cn/228043.htm
qnj.hjiocz.cn/559113.htm
qnj.hjiocz.cn/793393.htm
qnj.hjiocz.cn/600823.htm
qnj.hjiocz.cn/711753.htm
qnh.hjiocz.cn/642223.htm
qnh.hjiocz.cn/919913.htm
qnh.hjiocz.cn/357713.htm
qnh.hjiocz.cn/197113.htm
qnh.hjiocz.cn/826443.htm
