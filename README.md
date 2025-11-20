flowchart TD
    A[Manual Trigger<br>workflow_dispatch] --> B[Top Level Release Workflow]

    subgraph B [release.yml - Main Coordinator]
        B1[Setup Job] --> B2[Windows Sub-Workflow]
        B1 --> B3[Linux Sub-Workflow]
        B1 --> B4[Artifactory Upload Job]
        
        B2 --> B4
        B3 --> B4
    end

    subgraph B1 [Setup Job]
        B1A[Create Release Directory<br>with timestamp] --> B1B[Resolve TA Path<br>provided or integration_verified]
        B1B --> B1C[Set Output Variables<br>destination_path, ta_path, date]
    end

    subgraph B2 [windows-release.yml - Windows Pipeline]
        C1[Sign Files Job] --> C2[Build Installer Job]
        C2 --> C3[Sign MSI Job]
        C3 --> C4[Upload Signed Files to Artifactory]
    end

    subgraph C1 [Sign Files Job - Matrix: win]
        D1[Copy TA to Local<br>+ exclude_files_list.txt] --> D2[Run Signing Script<br>ta_sign.py]
        D2 --> D3[Copy Repacked Wheels<br>to install_files]
        D3 --> D4{Reversion Wheels?}
        D4 -- Yes --> D5[Run wheel_edit.py<br>Update versions]
        D5 --> D6[Update requirements.txt]
        D4 -- No --> D6
    end

    subgraph D2 [TA Signing Process - ta_sign.py]
        E1[Copy to Timestamped Dir<br>sgn_*_orig] --> E2[Find & Unpack<br>.whl files]
        E2 --> E3[Copy to NFS<br>for external signing]
        E3 --> E4[Trigger Jenkins<br>Signing Job]
        E4 --> E5[Monitor Jenkins Status<br>poll every 30s]
        E5 --> E6[Post-Sign Cleanup<br>Copy back from NFS]
        E6 --> E7[Repack Signed Wheels<br>to repacked_whls/]
    end

    subgraph C2 [Build Installer Job - Matrix: win]
        F1[Checkout Branches<br>current + installer] --> F2[Map R Drive<br>subst R: \\rdi\rdi]
        F2 --> F3[Build Custom Actions<br>C++ DLL & .NET]
        F3 --> F4[Build MSI with WiX<br>Package.wxs]
        F4 --> F5[Copy to<br>output_unsigned_msi/]
    end

    subgraph C3 [Sign MSI Job - Matrix: win]
        G1[Run Signing on<br>MSI/CAB files] --> G2[Rename MSI<br>xxx-lt-version.msi]
        G2 --> G3[Copy to<br>signed_installer/]
    end

    subgraph C4 [Upload Signed Files - Matrix: win]
        H1{Upload to Artifactory?}
        H1 -- Yes --> H2[Upload MSI to<br>installers-prod-local]
        H1 -- No --> H3[Skip Upload]
    end

    subgraph B3 [linux-release.yml - Linux Pipeline]
        I1[Build & Process Wheels] --> I2[Create .tgz Archive]
    end

    subgraph B4 [Artifactory Upload Job]
        J1{Upload to Artifactory?}
        J1 -- Yes --> J2[Upload .tgz + pip_list.txt<br>to release repository]
        J1 -- No --> J3[Skip Upload]
    end

    %% Final Outputs
    B2 --> K1[Windows Artifacts<br>Signed MSI + Wheels]
    B3 --> K2[Linux Artifacts<br>.tgz + Wheels]
    B4 --> K3[Artifactory<br>All Release Packages]
    
    K1 --> L[Release Complete]
    K2 --> L
    K3 --> L

    style A fill:#e1f5fe
    style L fill:#c8e6c9
    style B1 fill:#fff3e0
    style B2 fill:#e8f5e8
    style B3 fill:#e3f2fd
    style B4 fill:#f3e5f5
