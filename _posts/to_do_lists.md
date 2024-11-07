# TODO List

## SSR

## TAA

## 计算法线变换矩阵的另一种方法

## RenderTexture.memorylessMode

<!-- https://discussions.unity.com/t/rendertexture-memorylessmode/661299 -->

## VisibleLight.localToWorldMatrix 矩阵的第3列和第4列的意义

<!-- // For directional light, lightPos is a direction, and in light's local space, it's forward direction in local space is (0,0,1),
// after is multiplied by light's localToWorld matrix:
// localToWorldMatrix * (0,0,1,0), a direction has 0 in homogeneous coordinate;
// it returns the column 2 of the light's localToWorld matrix, and in lighting calculation in Shader,
// the light directional vector needs to point to light, so negative the direction here

// For point light and spot light, lightPos is a position in world space, it's original position in local space is (0,0,0),
// after is multiplied by light's localToWorld matrix:
// localToWorldMatrix * (0,0,0,1), a position has 1 in homogeneous coordinate;
// it returns the column 3 of the light's localToWorld matrix

// For spot light's direction, and in light's local space, it's forward direction in local space is (0,0,1),
// after is multiplied by light's localToWorld matrix:
// localToWorldMatrix * (0,0,1,0), a direction has 0 in homogeneous coordinate;
// it returns the column 2 of the light's localToWorld matrix, and in lighting calculation in Shader,
// the spot directional vector needs to point to light, so negative the direction here -->

## 球谐函数（Spherical Harmonics）

## VSM
