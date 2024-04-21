### 课程作业 加解密算法: chacha20 
```
chacha20 = "0.9.1"

// 加解密函数, 传入明文 输出密文, 传入密文,输出明文
// 算法需要的key长度为256位, 可以用摘要算法对输入的密码做摘要处理
pub fn process_text_crypto (
    mut buf: Vec<u8>,
    key: &[u8], // (ptr, length)
) -> Result<Vec<u8>> {
    let nonce = [0x0; 12];
    let key = blake3::hash(key);
    let mut cipher = ChaCha20::new(key.as_bytes().into(), &nonce.into());
    cipher.apply_keystream(&mut buf);
    Ok(buf)
}
```

### 课程作业 web服务加入对目录的浏览, 主要是对目录的遍历,和服务返回类型修改
```
use std::fs::read_dir;

返回类型调整:
(StatusCode, [(&'static str, &'static str); 1], String)

if p.is_dir() {
    let mut path = path.unwrap().replace("./", "/");
    if path.chars().last() != Some('/') {
        path.push('/');
    }
    match read_dir(&p)
        .unwrap()
        .map(|res| res.map(|e| {
            match e.path().file_name().map(|f|f.to_str()) {
                Some(Some(file)) => format!(r#"<li><a href="{}{}">{}</a></li>"#, path, file, file),
                    _ => "".into()
                }
            }
        ))
        .collect::<Result<Vec<_>, _>>()
    {
        Ok(files) => {
            (
                StatusCode::OK, 
                [("content-type", "html")], 
                format!(r#"<!doctype html><html><body><h1>{}</h1><ul>{}</ul></body></html>"#, path, files.concat())
            )
        }
        Err(e) => {
            warn!("Error reading dir: {:?}", e);
            (
                StatusCode::INTERNAL_SERVER_ERROR, 
                [("content-type", "text/plain")], 
                e.to_string()
            )
        }
    }
}
```

### 课程作业 json web token
```
jsonwebtoken = "9.3.0"
chrono = "0.4.38"

use chrono::{ Utc, Duration };
use jsonwebtoken::{
    decode, encode, Algorithm, DecodingKey, EncodingKey, Validation,
};

#[derive(Debug, Parser, PartialEq, Eq, Clone, serde::Serialize, serde::Deserialize)]
pub struct SignOpts {
    #[serde(skip)]
    #[arg(short, long)]
    pub secret: String, // 生成token的密钥密码

    #[arg(long)]
    pub sub: String,
    #[arg(long = "aud")]
    pub company: String,
    #[arg(long, value_parser = parse_exp_format)]
    pub exp: i64,
}

// 生成token
let header = Default::default();
let token = encode(&header, &self, &EncodingKey::from_secret(self.secret.as_bytes()))?;
println!("token {:?}", token);

// 解析token
let token = decode::<SignOpts>(&self.token, &DecodingKey::from_secret(self.secret.as_bytes()), &Validation::new(Algorithm::HS256))?;
println!("verify {:?}", token);

// 时间的解析, 支持天数
fn parse_exp_format(exp: &str) -> Result<i64, anyhow::Error> {
    
    let exp = exp.replace("d", "");
    let duration = Duration::days(exp.parse::<i64>()?);
    Ok(Utc::now().timestamp() + duration.num_seconds())
}
```
