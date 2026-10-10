storage/innobase/include/mtr0types.h　に mtr の種類が書いてある。

redo log の構造は、type(mtr_log_id_t),space_no,page_no,payload で構成されている。

mtr_t::added_rec() にブレイクポイントを張ってコールスタックを読んでいくとコードが読みやすい。
