# QDD-DETR
QDD-DETR: semantic P3–P5 queries; decoder samples stride-4 P2 and unpooled stride-2 stem S2. DVP applies affine projection after sparse sampling/aggregation; a sampled constant channel keeps bias at zero-padded edges, matching dense projection up to FP rounding. Pretrained-weight grafting initializes new levels with little detector disruption.
